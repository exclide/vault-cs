### boost::asio::io_context
Основной класс asio. Блокируется, пока есть асинхронные операции в очереди. Общается с операционной системой, использует select, poll, epoll, чтобы понимать, когда готовы сокеты к read/write - после получения от ОС сигнала о выполненной асинхронной операции, вызывает указанный хэндлер. Достает из очереди хэндлеры/калбэки, поставленные в эту очередь асинхронными операциями, async_accept, async_read, async_write

### boost::asio::ip::tcp::endpoint
ip:port, конструктор (ip, port); для ip можно использовать tcp::v4() - свой IP, boost::asio::ip::address::from_string()  

### boost::asio::ip::tcp::socket 
Абстракция над сокетом ОС, endpoint клиента и сервера, провод между двумя узлами

### boost::asio::ip::tcp::acceptor
Слушает эндпоинт и принимает соединения (listener, acceptor)  
Конструкторы:  
acceptor(ioc);  
acceptor(ioc, endpoint, true) - автоматически делает open, bind, listen, reuse_adr  
bind -> a.bind() - системный вызов к ОС на выдачу порта  
listen -> a.listen() - начинаем слушать сокет  
accept -> a.accept() - принять соединение  

### acceptor a(ioc, endpoint);  
Конструктор acceptor выше эквивалентен  
acceptor.open(endpoint.protocol());  
if (reuse_addr)  
acceptor.set_option(socket_base::reuse_address(true));  
acceptor.bind(endpoint);  
acceptor.listen();  
