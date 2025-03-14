# lab0--reliable byte stream

在开始 lab0 之前，我们还需要了解 CS144 中对于现代 C++的编码规范：

* 禁止裸指针，必要时只允许智能指针
* 禁止手动控制的动态内存分配
* 避免 C 风格字符串和强制类型转换方式（说的就是 string.h 里的字符串函数）
* 承接上条，只允许使用 std:: string 作为字符串对象，以及 C++ 的四种类型转换符
* 避免使用全局变量，缩小你的变量作用域和生命周期
* 使用 cmake --build build --target tidy 静态分析代码获取优化建议，使用 cmake --build build --target format 格式化代码



在 CS144 的 lab0 中，我们需要根据实验文档和框架代码完成一个有读端和写端的可靠字节流。框架代码在文件 `src/byte stream.hh` 和 `src/byte stream.cc` 中给出，我们在补充完整代码之后还需要进行测速。

![image-20250308151916020](https://raw.githubusercontent.com/QZQ54188/my_blog_img/main/image-20250308151916020.png)

接下来我们先看 `check0.pdf` 对于这个可靠字节流的要求：

* 字节流使用 `capacity` 进行初始化，但是字节流的长度是无限的，直到写端关闭或者读端读到了 `EOF`, 所以说初始化用的容量只是流在内存中可以储存的字节数
* 写端关闭写入流之后，不再接收任何字符的写入，而且在关闭写端时还需要再字节流中写入一个 EOF 标识符标志着字节流的结束
* 只要写端写入之后读端可以立即进行读取，文档中指出这里只有单线程，不需要考虑对象的读写抢占和加锁问题

接下来我们去分析给定的框架代码：

```c++
class ByteStream
{
public:
  explicit ByteStream( uint64_t capacity );

  // Helper functions (provided) to access the ByteStream's Reader and Writer interfaces
  Reader& reader();
  const Reader& reader() const;
  Writer& writer();
  const Writer& writer() const;

  void set_error() { error_ = true; };       // Signal that the stream suffered an error.
  bool has_error() const { return error_; }; // Has the stream had an error?

protected:
  // Please add any additional state to the ByteStream here, and not to the Writer and Reader interfaces.
};
```

框架代码中给定了一个基类 ByteStream，接着用这个基类派生了两个类 `Reader` 和 `Writer`，鉴于基类中给出了返回 `reader` 和 `writer` 对象的方法，我们不难猜出程序会在 `reader()` 和 `writer()` 方法中将基类做类型转换变为派生类。我们可以看 `byte_stream_helpers.cc` 文件中的实现，发现与我们的猜测一致：

```C++
Reader& ByteStream::reader()
{
  static_assert( sizeof( Reader ) == sizeof( ByteStream ),
                 "Please add member variables to the ByteStream base, not the ByteStream Reader." );

  return static_cast<Reader&>( *this ); // NOLINT(*-downcast)
}

const Reader& ByteStream::reader() const
{
  static_assert( sizeof( Reader ) == sizeof( ByteStream ),
                 "Please add member variables to the ByteStream base, not the ByteStream Reader." );

  return static_cast<const Reader&>( *this ); // NOLINT(*-downcast)
}

Writer& ByteStream::writer()
{
  static_assert( sizeof( Writer ) == sizeof( ByteStream ),
                 "Please add member variables to the ByteStream base, not the ByteStream Writer." );

  return static_cast<Writer&>( *this ); // NOLINT(*-downcast)
}

const Writer& ByteStream::writer() const
{
  static_assert( sizeof( Writer ) == sizeof( ByteStream ),
                 "Please add member variables to the ByteStream base, not the ByteStream Writer." );

  return static_cast<const Writer&>( *this ); // NOLINT(*-downcast)
}
```

这段代码中有两个需要注意的点。首先是使用 `static_assert` 进行检查：这个静态断言的目的是确保所有的成员变量都被定义在基类 `ByteStream` 中，而不是在派生类 `Reader` 或者 `Writer` 中。所有如果在派生类中添加了任何新的数据成员，那么派生类的大小就会大于基类，发生报错。正是有了这个设计模式，所以在我们接下来的开发中要求所有状态都必须存储在基类中，这样可以确保 `Reader` 和 `Writer` 共享相同的底层数据。

其次是使用 `static_cast` 将基类向派生类转化，按理来说这些转化是不安全的，应该使用 `dynamic_cast` 并且进行转化后的指针检查，再进行下一步流程，但是这里为什么直接使用 `static_cast` 从基类到子类进行转化？原因如下：

* 通过 `static_assert` 的保护，我们可以确保这个转换是安全的。因为基类和子类的大小一样，而类的普通成员函数不存在于类的实例中。
* 个人理解：`ByteStream` 类实际上是作为一个状态容器，而 `Reader` 和 `Writer` 是同一个对象的两个不同接口视图。

这整个设计的目的是实现一个字节流，它可以在同一个底层缓冲区上提供读取和写入的不同视图，并且保持数据的一致性，因为所有状态都保存在基类中。

接下来看派生类的代码：

```C++
class Writer : public ByteStream
{
public:
  void push( std::string data ); // Push data to stream, but only as much as available capacity allows.
  void close();                  // Signal that the stream has reached its ending. Nothing more will be written.

  bool is_closed() const;              // Has the stream been closed?
  uint64_t available_capacity() const; // How many bytes can be pushed to the stream right now?
  uint64_t bytes_pushed() const;       // Total number of bytes cumulatively pushed to the stream
};

class Reader : public ByteStream
{
public:
  std::string_view peek() const; // Peek at the next bytes in the buffer
  void pop( uint64_t len );      // Remove `len` bytes from the buffer

  bool is_finished() const;        // Is the stream finished (closed and fully popped)?
  uint64_t bytes_buffered() const; // Number of bytes currently buffered (pushed and not popped)
  uint64_t bytes_popped() const;   // Total number of bytes cumulatively popped from stream
};
```

正常来说如果我们想要传递一个对象类型，一般都会使用只读引用语义 const T&，或者说对于 std:: string 可以使用视图语义 std:: string_view；在 C++ 中，对象的传递使用值语义时一般都表示“在函数内部无论如何都会产生一次不可避免的复制”，或者是“对象必然需要在函数内部被修改，且修改操作与外界无关”。很显然，我们在管理传入的字节流时不需要对字节本身做任何修改，那么这只剩下一个可能：无论如何我们都需要将外界的 std:: string 拷贝并存储在我们自己的内部容器中。也就是说课程在要求我们存储外界传进来的 std:: string 对象本身。很明显，为了提高效率，我们需要对函数内左值对象 `std::string` 使用移动语义，避免没有意义的拷贝。

我们接下来分析需要我们实现的需求，这是一个可靠的字节流，意味着写端只要写入，读端就可以读取，或者读端一旦读取完毕，写端就可以再多写一个字符，除非已经被关闭。很像在一个无限长字节序列上滑动的窗口，被包围的部分就是能被读端读取到的字节，而且是一个FIFO的数据结构。

结合上述分析，我们可以选择使用 `string` 或者 `queue<string>` 实现，在分别使用两个数据结构实现之后，发现使用 `queue<string>` 的做法吞吐量要明显大于前者，特别是在进行多次小规模操作的时候，这是为什么呢：

* 首先 `std::string` 通常是一个连续的字符数组。当从字符串中 `pop` 数据时，可能需要将剩余的数据向前移动以填补被移除数据的空间。如果 `pop` 出的数据长度很小（例如每次只 `pop` 几个字节），但字符串中剩余的数据量很大，那么每次 `pop` 操作都需要将大量数据向前移动。这种数据移动的时间复杂度是 **O(n)**，其中 `n` 是剩余数据的长度。
* `std::string` 在动态调整大小时可能会触发内存重新分配和复制操作。虽然现代 `std::string` 实现通常会预留额外空间以避免频繁重新分配，但在极端情况下仍然可能发生。
* 队列的设计使得 对字节流的 `pop` 与 `push` 操作只需要移除队头元素，而不需要移动剩余数据。

`std::string_view` 是 C++17 引入的一个轻量级非拥有（non-owning）字符串视图类。它提供了一种高效的方式来访问字符串数据，而无需复制或管理内存。它的主要特点包括：

- **非拥有语义**：`std::string_view` 不拥有字符串数据，它只是对现有字符串的引用。
- **高效**：由于不需要复制数据，`std::string_view` 的构造和复制成本很低。
- **只读**：`std::string_view` 是只读的，不能通过它修改底层的字符串数据。

补充完一些细节之后，我们就可以开始代码的实现了。由于测试样例给的相对丰富，所以代码编写的难度其实不是很高，这是我的[代码实现](https://github.com/QZQ54188/minnow)，附上一张测试图。

![image-20250313192003129](https://raw.githubusercontent.com/QZQ54188/my_blog_img/main/image-20250313192003129.png)



