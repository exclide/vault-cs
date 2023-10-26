### boost::asio::io_context
Основной класс asio. Блокируется, пока есть асинхронные операции в очереди. Общается с операционной системой, использует select, poll, epoll, чтобы понимать, когда готовы операции read/write/accept - после получения от ОС сигнала о выполненной асинхронной операции, вызывает указанный хэндлер. Достает из очереди хэндлеры/калбэки, поставленные в эту очередь асинхронными операциями, async_accept, async_read, async_write

Есть несколько способов выполнения:
1. Один ioc на один поток. Самый простой и безопасный.
2. Один ioc на несколько потоков. Запускаем ioc.run() в разных потоках.
3. Несколько ioc на несколько потоков. Создаем несколько ioc и каждый запускаем в своем потоке.


### boost::asio::ip::tcp::endpoint
ip:port, конструктор (ip, port); для ip можно использовать tcp::v4() - свой IP, boost::asio::ip::address::from_string()  

Пример подключения клиента к эндпоинту:  
```cpp
using boost::asio::io_context;
using boost::asio::ip::tcp;  
using boost::asio::ip::address;

io_context ioc;
tcp::socket s(ioc);
tcp::endpoint endpoint(address::from_string(argv[1]), std::stoi(argv[2]));
s.connect(endpoint);
```

### boost::asio::ip::tcp::socket 
Абстракция над сокетом ОС, endpoint клиента и сервера, провод между двумя узлами

### boost::asio::ip::tcp::acceptor
Слушает эндпоинт и принимает соединения (listener, acceptor)  
Конструкторы:  
acceptor(ioc);  
acceptor(ioc, endpoint, true) - автоматически делает open, bind, listen, reuse_adr  
open -> a.open(protocol) - открывает сокет с указанным протоколом (tcp/udp)
bind -> a.bind(endpoint) - привязывает сокет к указанному local endpoint (IP:Port)
listen -> a.listen() - начинаем слушать сокет  
accept -> a.accept() - принять соединение, вернув сокет с парой endpoint, связанным с конкретным соединением

### acceptor a(ioc, endpoint);  
Конструктор acceptor выше эквивалентен  
acceptor.open(endpoint.protocol());  
if (reuse_addr)  
acceptor.set_option(socket_base::reuse_address(true));  
acceptor.bind(endpoint);  
acceptor.listen();  

Пример сервера на прослушивание и прием соединений с заданного порта:  
```cpp
using boost::asio::io_context;  
using boost::asio::ip::tcp;  

io_context ioc;
tcp::endpoint endpoint(tcp::v4(), std::stoi(argv[1]));
tcp::acceptor acceptor(ioc, endpoint); //open, bind, listen
tcp::socket socket(ioc);
acceptor.accept(socket); //блокируется до подключения к сокету
```


### Функции/методы для записи/чтения сокета
Методы:  
1. async_read_some(buffer, handler) - читает хотя бы 1 байт в буффер, после вызывает хэндлер. Возвращает количество прочитанных байтов
2. async_write_some(buffer, handler) - пишет хотя бы 1 байт в буффер, после вызывает хэндлер. Возвращает количество записанных байтов

Функции:
1. async_read(socket, buffer, handler) - закончит только когда прочитает все байты в буффер, состоит из операций async_read_some
2. async_read_until(socket, buffer, delimiter, handler) - прочитает до первого delimiter
3. async_write(socket, buffer, handler) - закончит только когда запишет все байты буффера в сокет

Пример использования async_read_until с хэндлером-лямбдой:  
```cpp
void DoRead() { //захватим shared_ptr самого себя, чтобы продлжить время жизни объекта
	auto self{shared_from_this()};

	boost::asio::async_read_until(socket, buf, "\n",
	[this, self] (error_code& err, std::size_t bytesRead) {
		if (!err) {
			DoWrite(bytes);
		}
	});
}
```

### boost::asio::buffer()
Обертка над данными, которые читаем/пишем из сокета. Можно быть std::string, void* и размер в байтах, char[], std::vector из POD типов - plain old data, то есть моделек

Пример записи в сокет из буффера:  
```cpp
std::string data = "data";
boost::asio::async_write(socket, boost::asio::buffer(data), handler);
```

### boost::asio::streambuf


### ioc.post(handler), ioc.dispatch(handler)
Отправляет кастомный хандлер в очередь на выполнение io_context;   
**post** - вернется сразу, исполнит handler после  
**dispatch** - может исполнить хэндлер сразу, если текущий поток вызвал ioc.run()  

### io_context::strand(ioc)
Если у нас несколько потоков выполняют ioc.run(), то теряется последовательный порядок выполнения handler-функций, могут возникнуть гонки. Если мы хотим избежать гонок и добиться последовательного выполнения хэндлеров, нам нужно постить (post(), dispatch()) хэндлеры в strand.

Strand - это executor, то есть политика/стратегия выполнения handler-функций в очереди потоками. По дефолту стратегия - system executor. 

boost::asio::make_strand(ioc) - вернет strand для контекста ioc.  
Пример использования с 3 потоками:  
```cpp
io_context ioc;
auto strand = boost::asio::make_strand(ioc);

for (int i = 0; i < 10; i++) {  
boost::asio::post(strand,[i]() { std::cout << i << " " << i << std::endl; });  
//boost::asio::post(ioc, [i]() { std::cout << i << " " << i << std::endl; }); //гонка 
}  
  
std::vector<std::thread> threads;  
for (int i = 0; i < 2; i++) {  
	threads.emplace_back([&ioc]() { ioc.run(); });  
}  
  
for (auto& t : threads) {  
	t.detach();  
}  
  
ioc.run();
```

Как видно, если постить в strand, порядок выводов сохраняется.  

Под каждый сокет/соединение/сессию, мы будем заводить свой strand, таким образом избежав ситуаций, где в один сокет пишет более одного потока.