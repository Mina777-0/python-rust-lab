# TCP and SSL

    class SocketHandler:
        def __init__(self):
        self.ssock: Optional[socket.socket]= None
        self.loop= asyncio.get_running_loop()

        def connect(self, host:str, port:int):

        context= ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)

        try:
            context.load_cert_chain(certfile=cert_file, keyfile=key_file, password=password_bytes)
        except FileNotFoundError as e:
            raise e

        sock= socket.socket(family=socket.AF_INET, type=socket.SOCK_STREAM)
        sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        sock.setblocking(False)
        sock.bind((host, port))
        sock.listen(1)
        print(f"\n[SERVER]: Server is listening on {host}:{port}")

        try:
            conn, addr= sock.accept()
            print(f"\n[SERVER]: Unencrypted connection to {addr}")

            self.ssock= context.wrap_socket(conn, server_side= True) 
            print(f"\n[SERVER]: SSL Handshake is complete. Protocol: {self.ssock.version()}")
            
        except ConnectionAbortError as e:
            raise e 
        except BrokenPipeError as e:
            raise e 
        except Exception as e:
            raise e 

        async def handle_connection(self):
        if self.ssock is None:
            print(f"\n[SERVER]: Connection is closed or unestablished.")
            return 



        try:
            while True:
                try:
                    
                    # Use the native recv_into. This performs decryption in-place into the buffer

                except (ssl.SSLWantWriteError, ssl.SSLWantReadError):
                    # IF the socket is empty, wait for it to be ready again
                    # We yield control back to the loop to talk to the epoll or kqueue of the kernel unitl the FD is ready
                    waiter= self.loop.create_future()
                    fd= self.ssock.fileno()
                    
                    self.loop.add_reader(fd, lambda: waiter.done() or waiter.set_result(None))
                    try:
                        await waiter
                    finally:
                        self.loop.remove_reader(fd)
                    continue

        except ConnectionResetError:
            raise
        except Exception as e:
            raise e


* With a regular TCP socket, if there’s no data, you get a BlockingIOError. But SSL is a layer on top of TCP. To give you one byte of plaintext,
it might need to read a 16KB encrypted block from the wire.

* SSLWantReadError: The SSL engine is saying, "I have some encrypted data, but not enough to finish a full 'record' to decrypt it for you. 
Tell me when the hardware has more bytes for me."

* SSLWantWriteError: (The weird one) Sometimes, while you are trying to read, SSL needs to write (like a handshake renewal). 
It’s saying, "I can't give you data until I send this encrypted packet out first."

* Since this is an async function, we can’t just sit in a while True loop checking the socket. That would pin your CPU at 100% (busy-waiting).
Instead, we create a Future. Think of this as a "promise" that we will eventually wake this function back up.

* This is where the performance happens. You are telling the operating system’s kernel (epoll on Linux, kqueue on macOS):
"Hey, keep an eye on this File Descriptor (fd). The second the network card receives new data for it, execute this little lambda function."
The lambda simply checks if the waiter is already finished; if not, it calls set_result(). This "completes" the promise.

** When a process starts, the OS automatically provides it with three "streams": Standard Input (keyboard), Standard Output (screen), and Standard Error.
stdin : 0
stdout: 1
stderr: 2

If it happens and a socket is opened, the OS provide a new file discriptor with number 3 which handles stdin, stdout, stderr
if it happend and use open('data.txt'), the OS provide a new file discriptor with number 4 which handles stdin, stdout, stderr
Why only one?
Unlike the "Standard" setup (where input and output are separated into 0 and 1), a single FD for a file or a socket is bi-directional at the kernel level:
- Files: You use the same FD to read() from a offset and write() to an offset.
- Sockets: A socket FD represents a full-duplex connection. You send data and receive data through that same integer.

A File Descriptor is an unsigned integer that the OS uses as an index to track an open resource.

Think of the OS as a giant librarian. When you open a file or a socket, the librarian doesn't give you the "book" (the actual data/hardware); 
they give you a ticket number (the FD). When you want to read or write, you show that ticket number.
0: Standard Input (stdin)
1: Standard Output (stdout)
2: Standard Error (stderr)
3 and above: Your files, database connections, and sockets.

What is fileno()?
In Python, socket objects are high-level "wrappers." They contain a lot of Python-specific logic. However, the OS kernel doesn't understand 
Python objects; it only understands those integer ticket numbers.
The .fileno() method is how you ask the Python object: "Hey, what is the actual integer ID the operating system gave you for this connection?"

What is add_reader()?
This is a core method of the asyncio event loop. It is the bridge between Python's Task system and the OS's I/O Multiplexing 
(like epoll or kqueue).

The definition:
loop.add_reader(fd, callback, *args)

It tells the event loop: "Monitor this File Descriptor. The moment the OS says there is data waiting to be read on this FD, stop what 
you are doing and run this callback function immediately."
                    
The loop.add_reader() method is a low-level, callback-based API in Python's asyncio library used to watch a file descriptor (FD) for read 
availability. When data is available to be read on the FD (e.g., a socket), the provided callback function is invoked.

The add_writer() method in Python's asyncio library is a low-level event loop method used to register a callback function to be executed when 
a specific file descriptor (FD) or socket is ready for writing