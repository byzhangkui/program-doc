# WebRTC的指针内存管理

## 引用计数原则
- 对象的初始引用计数是1。
- 当引用被创建或者拷贝，引用计数加1。
- 当对象的引用被销毁或者重新赋值，对象的引用计数减1。
- 当引用计数减为0时，对象被销毁。
- 多线程场景下，引用计数的增减应当是原子操作。

## scoped_refptr和RefCountedObject
### scoped_refptr
```C++
template <class T>
class scoped_refptr {
 public:
  //1.默认构造函数
  scoped_refptr() : ptr_(nullptr) {}

  //2.构造函数
  scoped_refptr(T* p) : ptr_(p) {
    if (ptr_)
      ptr_->AddRef();
  }

  //3、拷贝构造函数
  scoped_refptr(const scoped_refptr<T>& r) : ptr_(r.ptr_) {
    if (ptr_)
      ptr_->AddRef();
  }

  //4、带隐式类型转换的拷贝构造函数
  template <typename U>
  scoped_refptr(const scoped_refptr<U>& r) : ptr_(r.get()) {
    if (ptr_)
      ptr_->AddRef();
  }

  //5、Move构造函数.
  scoped_refptr(scoped_refptr<T>&& r) : ptr_(r.release()) {}

  //6、带隐式类型转换的Move构造函数.
  template <typename U>
  scoped_refptr(scoped_refptr<U>&& r) : ptr_(r.release()) {}

  //7、析构函数
  ~scoped_refptr() {
    if (ptr_)
      ptr_->Release();
  }

  //8、获取原始指针
  T* get() const { return ptr_; }
  //9、重载*
  operator T*() const { return ptr_; }
  //10、重载->
  T* operator->() const { return ptr_; }

  //11、释放指针所有权，返回值是该对象当前保存的指针。该方法返回后，此对象不再拥有之前的对象，而是拥有一个空指针。
  T* release() {
    T* retVal = ptr_;
    ptr_ = nullptr;
    return retVal;
  }

  //12、赋值函数
  scoped_refptr<T>& operator=(T* p) {
    // AddRef first so that self assignment should work
    if (p)
      p->AddRef();
    if (ptr_ )
      ptr_ ->Release();
    ptr_ = p;
    return *this;
  }
  //13、赋值函数
  scoped_refptr<T>& operator=(const scoped_refptr<T>& r) {
    return *this = r.ptr_;
  }
  //14、带隐式类型转换的赋值函数
  template <typename U>
  scoped_refptr<T>& operator=(const scoped_refptr<U>& r) {
    return *this = r.get();
  }
  //15、Move赋值
  scoped_refptr<T>& operator=(scoped_refptr<T>&& r) {
    scoped_refptr<T>(std::move(r)).swap(*this);
    return *this;
  }
  //16、带隐式类型转换的Move赋值
  template <typename U>
  scoped_refptr<T>& operator=(scoped_refptr<U>&& r) {
    scoped_refptr<T>(std::move(r)).swap(*this);
    return *this;
  }

  //17、swap
  void swap(T** pp) {
    T* p = ptr_;
    ptr_ = *pp;
    *pp = p;
  }

  //18、swap
  void swap(scoped_refptr<T>& r) {
    swap(&r.ptr_);
  }

 protected:
  T* ptr_;
};
```

> scoped_refptr实现了基于引用计数的内存管理，实际使用中同RefCountedObject配合来实现对指针对象的内存管理。

### RefCountedObject
```C++
class RefCountInterface {
 public:
  virtual int AddRef() const = 0;
  virtual int Release() const = 0;

 protected:
  virtual ~RefCountInterface() {}
};
```
> RefCountInterface定义引用计数接口

```C++
template <class T>
class RefCountedObject : public T {
 public:
  RefCountedObject() {}

  template <class P0>
  explicit RefCountedObject(P0&& p0) : T(std::forward<P0>(p0)) {}

  template <class P0, class P1, class... Args>
  RefCountedObject(P0&& p0, P1&& p1, Args&&... args)
      : T(std::forward<P0>(p0),
          std::forward<P1>(p1),
          std::forward<Args>(args)...) {}

  virtual int AddRef() const { return AtomicOps::Increment(&ref_count_); }

  virtual int Release() const {
    int count = AtomicOps::Decrement(&ref_count_);
    if (!count) {
      delete this;
    }
    return count;
  }

  virtual bool HasOneRef() const {
    return AtomicOps::AcquireLoad(&ref_count_) == 1;
  }

 protected:
  virtual ~RefCountedObject() {}

  mutable volatile int ref_count_ = 0;
};
```
> RefCountedObject实现基于原子锁的引用计数管理。


## 优点：
1.线程安全的引用计数。  
2.析构函数是protected，避免显式delete。

## 示例
```C++
class PeerConnectionFactoryInterface : public RefCountInterface {
public:
	virtual void Show() = 0;
};

class PeerConnectionFactory : public PeerConnectionFactoryInterface {
public:
	virtual void Show() override {  }
};

class PeerConnectionSubFactory : public PeerConnectionFactory {
public:
	virtual void Show() override  {  }
};
```

### scoped_refptr
```C++
//1、默认构造函数
scoped_refptr<PeerConnectionFactory> pc_factory_def;

//2、构造函数
scoped_refptr<PeerConnectionFactory> pc_factory(
  new RefCountedObject<PeerConnectionFactory>());

//3、拷贝构造函数
scoped_refptr<PeerConnectionFactory> pc_factory2(pc_factory);

//4、带隐式类型转换的拷贝构造函数
scoped_refptr<PeerConnectionSubFactory> pc_sub_factory(new RefCountedObject<PeerConnectionSubFactory>());
scoped_refptr<PeerConnectionFactory> pc_factory3(pc_sub_factory);

//5、Move构造函数.
scoped_refptr<PeerConnectionFactory> pc_factory4(std::move(pc_factory3));

//6、带隐式类型转换的Move构造函数.
scoped_refptr<PeerConnectionFactory> pc_factory5(std::move(pc_sub_factory));

//8、获取原始指针
pc_factory5.get()->Show();
//9、重载*
(*pc_factory5).Show();
//10、重载->
pc_factory5->Show();

//11、释放指针所有权
PeerConnectionFactory* p = pc_factory5.release();


//12、赋值函数
scoped_refptr<PeerConnectionFactory> pc_factory6 = new RefCountedObject<PeerConnectionFactory>();
//13、赋值函数
pc_factory_def = pc_factory6;
//14、带隐式类型转换的赋值函数
scoped_refptr<PeerConnectionSubFactory> pc_factory8 = new RefCountedObject<PeerConnectionSubFactory>();
pc_factory_def = pc_factory8;
//15、Move赋值
pc_factory_def = std::move(pc_factory6);
//16、带隐式类型转换的Move赋值
pc_factory_def = std::move(pc_factory8);
//18、swap，会调用17
pc_factory.swap(pc_factory_def);
```
