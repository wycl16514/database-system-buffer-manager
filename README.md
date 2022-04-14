数据库系统设计中一个必须关注的瓶颈就是读写效率。由于数据库系统要处理高吞吐量的数据读写，由于数据量大，系统不能总是把所有数据都存储在内存中，但是频繁的操作磁盘就会导致系统效率大大降低，因此我们必须要有办法权衡数据在内存和磁盘上的存储， 我们要让数据尽可能多的从内存进行读写，尽可能少的触发磁盘操作，因此设计一个有效的缓存管理系统对效率有致命的作用。

我们本节要设计一个缓存管理器，它会预先分配固定数量的内存页，也就是我们前面几节实现的Page对象，由此形成一个内存池，当其他组件想要读写数据时，他们先通过缓存管理器获得内存页，然后必须在给定协议的基础上与缓存管理器进行交互，这样才能确保后者有效的保证数据的准确性和读写的效率性，这里有几个要点需要说明：

1， 如果一个缓存页被分配给其他组件进行读写，那么这个缓存页就会处于一个状态叫"pinned",意思就是有个钉子把这个页面给钉住了。只要一个缓存页处于"pinned"状态，缓存管理器就不能对其进行回收。

2，如果缓存页使用完毕，使用它的客户必须对它进行"unpin",这点就像c++中内存分配的new和delete，如果不unpin就会导致可用缓存遗失。

3.当某个组件向缓存管理器申请缓存页面读取某个二进制文件对应的区块时，可能会有两种情况发送，一种情况是要读取的数据已经存储在某个页面里，这样缓存管理器就会设置这个页面为pinned状态，然后将页面提交给客户，等客户完成使用后让管理器对该页面执行unpin操作，这里需要注意的是，一个已经被pin过的页面可以再次被pin，这意味着同一个缓存页面有多高客户在读取，管理器会对页面设置一个pin计数，当页面被pin时计数加1，被unpin时计数减1，当计数为0时说明该页面没有再被使用，可以分配给其他数据。

第二种情况是数据还没有读入缓存，同时管理器还有空闲的内存页面可以使用，这样管理器将数据读入空闲页面，然后将页面提交给客户。

第三种情况是数据没有读入内存，但是管理器已经没有空闲页面可用，此时客户可能就得等待。

这里需要注意的还有，当多个客户在使用同一个内存页面时，他们都可以根据自己的需要读写页面，缓存管理器不在乎数据的一致性，这些需要我们在后面实现的并发管理器来保证。为了保证效率，当一个页面被客户使用完后，如果里面数据有修改，管理器也不会直接将数据写入磁盘，如果接下来有其他客户同样需要读写相同的区块数据，那么管理器会直接将页面提交给客户，于是第二个客户的读写有可能会覆盖第一个客户的读写数据，只有在合适的时机管理器才会将页面数据写入磁盘，这个我们后面会解释。

如果客户要读取的数据还没有读入某个缓存页面，那么管理器会执行的操作如下：首先它会选择一个处于unpin状态的页面，这里又有两种情况需要考虑，首先是这个页面原先读取了其他区块的数据，而且这些数据被其他客户进行了修改，那么管理器必须要先将该页面的数据写入对应的文件区块，然后才能将新区块的数据写入该页面；如果这个页面的数据只读但没有被写入，那么管理器就能直接将所需区块的数据写入该页面。

为了实现缓存管理器，我们需要实现一个Buffer对象，它实际上是在我们前面实现的Page基础上再增加一些管理计数功能，例如一个页面被pin的次数，并负责将计数为0的page重新放入缓存池等。同时Buffer对象也负责Page针对磁盘的写入，当一个缓存页使用计数为0时，Buffer会根据策略延迟将其数据写入磁盘，因为这个页面的数据在后面可能又被其他客户或组件需要，那么Buffer会直接将这个页面提交给客户，于是客户就能直接读取已经被修改后的数据。

这里我们还需要考虑一个死锁问题，假设缓存只有两个页面编号为1，和2，此时来了两个客户A,B他们都需要同时请求两个页面，如果页面1被分配给了A,页面2分配给了客户B,那么客户A就会一直等待页面2，客户B就会一直等待页面1，于是就陷入了死锁状态，我们处理这种情况的做法是增加超时机制，如果客户在申请时没有可用的页面，那么就进入等待状态，如果等待时间过长，例如10秒，那么缓存管理器给客户返回一个错误，然后客户自行将已经申请的页面进行释放，然后再次进入申请操作。

Buffer只有在两种情况下才会将缓存页面的数据写入磁盘，一种是当前页面需要读取其他区块的数据，第二种情况是它相应的写数据接口被调用时。另外我们还需要关注的是缓存策略，假设当前我们只有3个缓存页，而且这些页面已经有数据读取，此时来了第4个请求，那么我们需要将3个页面中的数据写入磁盘，然后读取新数据，那么我们如何选取哪个页面呢，这里我们使用LRU缓存策略，也就是换取那个最长时间没有读写的页面，下面我们看看具体代码实现，首先在项目中增加一个子目录叫buffer_manager,然后在里面增加一个文件叫buffer.go，实现代码如下：
```
package buffer_manager

import (
	fm "file_manager"
	log "log_manager"
)

type Buffer struct {
    fm  *fm.FileManager
    lm  *log.LogManager
	contents *fm.Page  //用于存储磁盘数据的缓存页面
	blk   *fm.BlockId
	pins  uint32 //被引用计数
	txnum  int32 //交易号，暂时忽略其作用
	lsn    int32 //对应日志号，暂时忽略其作用
}

func NewBuffer(fm *fm.FileManager, lm *lm.LogManager) *Buffer {
	return &Buffer{
		fm : fm,
		lm: lm,
		contents: fm.NewPageBySize(fm.BlockSize()),
	}
}

func (b *Bufffer) Contents() *fm.Page {
	return b.contents 
}

func (b *Buffer)Block() *fm.BlockId {
	return b.blk 
}

func (b *Buffer)SetModified(int32 txnum, int32 lsn) {
	//如果客户修改了页面数据，必须调用该接口通知Buffer
	b.txnum = txnum 
	if lsn > 0 {
		b.lsn = lsn 
	}
}

func (b *Buffer)IsPinned() bool {
	return b.pins > 0
}

func (b *Buffer) ModifyingTx() int32 {
	return b.txnum
}

func (b *Buffer) AssignToBlock(block *fm.BlockId) {
	//将当前页面分发给其他区块
	b.Flush() //当页面分发给新数据时需要判断当前页面数据是否需要写入磁盘
	b.blk = block
	b.fm.Read(b.blk, b.Contents()) //将对应数据从磁盘读取页面
	b.pins = 0 
}

func (b *Buffer) Flush() {
	if b.txnum >= 0 {
		//当前页面数据已经被修改过，需要写入磁盘
		lm.FlushByLSN(b.lsn) //先将修改操作对应的日志写入
		fm.Write(b.blk, b.Contents()) //将数据写入磁盘
		b.txnum = -1
	}
}


func (b *Buffer) Pin() {
	b.pins = b.pins + 1
}

func (b *Buffer) Unpin() {
	b.pins = b.pins - 1
}

```
接下来我们看看缓存管理器的实现代码，增加一个buffer_manager.go，实现代码如下：
```
package buffer_manager

import (
	fm "file_manager"
	lm "log_manager"
	"sync"
	"errors"
	"time"
)

const (
	MAX_TIME = 3 //分配页面时最多等待3秒
)

type BufferManager struct {
	buffer_pool []*Buffer
	num_available uint32
	mu sync.Mutex 
}

func NewBufferManager(fm *fm.FileManager, lm *lm.LogManager, num_buffers uint32) *BufferManager {
    buffer_manager := &BufferManager{
		num_available: num_buffers,
	}
	for i := uint32(0); i < num_buffers; i++ {
		buffer := NewBuffer(fm, lm)
		buffer_manager.buffer_pool = append(buffer_manager.buffer_pool, buffer)
	}

	return buffer_manager
}

func (b *BufferManager) Available() uint32 {
	//当前可用缓存页面数量
	b.mu.Lock()
	defer b.mu.Unlock()
	return b.num_available
}

func (b *BufferManager) FlushAll(txnum int32) {
	b.mu.Lock()
	defer b.mu.Unlock()
	//将给定交易的读写数据全部写入磁盘
	for _, buff := range b.buffer_pool {
		if buff.ModifyingTx() == txnum {
			buff.Flush()
		}
	}
}

func (b *BufferManager) Pin(blk *fm.BlockId) (*Buffer, error) {
	b.mu.Lock()
	defer b.mu.Unlock()

    start := time.Now()
	buff := b.tryPin(blk)
	for buff == nil && b.waitingTooLong(start) == false {
		//如果无法获得缓存页面，那么让调用者等待一段时间后再次尝试
		time.Sleep(MAX_TIME * time.Second)
		buff = b.tryPin(blk)
		if buff == nil {
			return nil, errors.New("No buffer available , cafule for dead lock")
		}
	}

	return buff, nil 
}

func (b *BufferManager) Unpin(buff *Buffer) {
	b.mu.Lock()
	defer b.mu.Unlock()

	if buff == nil {
		return 
	}

	buff.Unpin()
	if !buff.IsPinned() {
		b.num_available = b.num_available + 1 
		//notifyAll() //唤醒所有等待它的线程,等到设计并发管理器时再做处理
	}
}

func (b *BufferManager) waitingTooLong(start time.Time) bool{
	elapsed := time.Since(start).Seconds()
	if elapsed >= MAX_TIME {
		return true
	}

	return false
}

func (b *BufferManager) tryPin(blk *fm.BlockId) *Buffer {
	//首先看给定的区块是否已经被读入某个缓存页
	buff := b.findExistingBuffer(blk)
	if buff == nil {
		//查看是否还有可用缓存页，然后将区块数据写入
		buff = b.chooseUnpinBuffer()
		if buff == nil {
			return nil 
		}
		buff.AssignToBlock(blk)
	}

	if buff.IsPinned() == false {
		b.num_available = b.num_available - 1
	}

	buff.Pin()
	return buff
}

func (b *BufferManager) findExistingBuffer(blk *fm.BlockId) *Buffer{
	//查看当前请求的区块是否已经被加载到了某个缓存页，如果是，那么直接返回即可
	for _, buffer := range b.buffer_pool {
		block := buffer.Block()
		if block != nil && block.Equal(blk) {
			return buffer 
		}
	}

	return nil 
}

func (b *BufferManager) chooseUnpinBuffer() *Buffer {
	//选取一个没有被使用的缓存页
	for _, buffer := range b.buffer_pool {
		if !buffer.IsPinned() {
			return buffer 
		}
	}

	return nil 
}
```
当前我们设计的缓存策略比较简单，先从缓存池中查看是否还有可用页面，如果没有那么就先等待一段时间再看看，如果等待后还是没有，那么我要警惕出现死锁情况，此时我们返回错误，收到错误的客户或外部组件把自己当前获得的页面先释放，然后再发起请求看看。目前我们只考虑了单线程情况，在后续设计中，为了提高吞吐量和系统运行效率，我们必然要使用并发，所以后续我们还会有并发管理器，到时候我们还会对代码进行修改，最后通过测试看看当前实现的逻辑是否正确，增加buffer_manager_test.go，然后实现如下代码:
```
package buffer_manager

import (
	fm "file_manager"
	"github.com/stretchr/testify/require"
	lm "log_manager"
	"testing"
)

func TestBufferManager(t *testing.T) {
	file_manager, _ := fm.NewFileManager("buffertest", 400)
	log_manager, _ := lm.NewLogManager(file_manager, "logfile")
	bm := NewBufferManager(file_manager, log_manager, 3)

	buff1, err := bm.Pin(fm.NewBlockId("testfile", 1)) //这块缓存区在后面会被写入磁盘
	require.Nil(t, err)

	p := buff1.Contents()
	n := p.GetInt(80)
	p.SetInt(80, n+1)
	buff1.SetModified(1, 0) //这里两个参数先不要管

	buff2, err := bm.Pin(fm.NewBlockId("testfile", 2))
	require.Nil(t, err)
	_, err = bm.Pin(fm.NewBlockId("testfile", 3))
	require.Nil(t, err)
	//下面的pin将迫使缓存管理区将buff1的数据写入磁盘
	_, err = bm.Pin(fm.NewBlockId("testfile", 4))
	//由于只有3个缓存页，这里的分配要返回错误
	require.NotNil(t, err)

	bm.Unpin(buff2)
	buff2, err = bm.Pin(fm.NewBlockId("testfile", 1))
	require.Nil(t, err)

	p2 := buff2.Contents()
	p2.SetInt(80, 9999)
	buff2.SetModified(1, 0)
	bm.Unpin(buff2) //注意这里不会将buff2的数据写入磁盘

	//将testfile 的区块1读入，并确认buff1的数据的确写入磁盘
	page := fm.NewPageBySize(400)
	b1 := fm.NewBlockId("testfile", 1)
	file_manager.Read(b1, page)
	n1 := page.GetInt(80)
	require.Equal(t, n, n1)
}

```
上面的测试代码可以顺利通过，通过测试代码逻辑，我们可以更加容易掌握缓存管理的设计逻辑。[代码下载地址](https://gitee.com/coding-disney/write-your-own-database-system.git)：https://gitee.com/coding-disney/write-your-own-database-system.git，[更多有趣内容点这里](http://m.study.163.com/provider/7600199/index.htm?share=2&shareId=7600199)：http://m.study.163.com/provider/7600199/index.htm?share=2&shareId=7600199
