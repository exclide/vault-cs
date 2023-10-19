boost::asio::ip::tcp::socket - абстракция над сокетом ОС

### Server 
boost::asio::ip::tcp::acceptor a;  
bind -> a.bind() - системный вызов к ОС на выдачу порта
listen -> a.listen() - начинаем слушать сокет
accept -> a.accept() - принять соединение  

### Client
connect -> boost::asio::connect