---
layout: post
title: "Unit testing Paramiko"
date: 2023-03-25 23:34:26 +0000
categories: unit tests, paramiko, python
---

This post came about while writting unit tests for a project that uses [Paramiko](https://www.paramiko.org/), a python library for running in-python ssh commands. For context, in this project a user can upload a file to a server, and the server will run a script on the file. The script is run on the server, not on the user's computer. Communication between the user and the server, i.e. running commands on the server, uploading and downloading files, is done through ssh using paramiko.

For this purpose i created a context manager responsible for establishing, and closing, the connection to the server. This worked as follows:

{% highlight python %}

class ConnectorParamiko(Connector):

    def __init__(self, config_file: str) -> None:
        super().__init__()
        self.prep_input(config_file)
        self.connect()

    def connect(self):

        paramiko_rsa_key = paramiko.RSAKey.from_private_key_file(
            self.rsa_key_path)
        self.rsa_key = paramiko_rsa_key

        self.conn = paramiko.SSHClient()
        self.conn.set_missing_host_key_policy(paramiko.AutoAddPolicy())
        self.test_connection()


    def __enter__(self):

        try:

            self.conn.connect(
                hostname=f"{self.ip_address}",
                username=f"{self.username}",
                pkey=self.rsa_key
            )

        except paramiko.ssh_exception.SSHException as error:
            print("SSH connection error")
            sys.exit(1)

        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.conn.close()

{% endhighlight %}

In the code above, `self.conn` is an instance of `paramiko.SSHClient()`. The `__enter__` method establishes the connection, and the `__exit__` method closes it.

Other methods of `ConnectorParamiko` depend on Paramiko methods, such as `self.conn.exec_command()`, `self.conn.put()`, `self.conn.get()`, etc.

And this works fine, until you try to write unit tests for it. First you need to mock the `paramiko.SSHClient()` object, and you need to mock the methods that depend on it. And you need a way to mock a remote server. The package [`MockSSH`](https://pypi.org/project/MockSSH/), built on top of `paramiko`, has been created for this exact purpose. It allows you to create a mock server, and to mock the methods of `paramiko.SSHClient()`. But `MockSSH` is a context manager, and so is `ConnectorParamiko`, so how do you use them together to test the methods of `ConnectorParamiko`?

After some research, I found a solution that works. It turns out you can have nested context managers. So i could yield the `MockSSH` context manager from the `ConnectorParamiko` context manager, and then use the `MockSSH` context manager to mock the methods of `paramiko.SSHClient()`:

{% highlight python %}

    def connect(self):
        """
        create local ssh mock server"""
        users = {
            "test-user": "/home/bioinf/.ssh/id_rsa",
        }
        self.server = mockssh.Server(users)

    def __enter__(self):
        self.server.__enter__()
        with self.server as s:
            self.conn = s.client("test-user")
            return self

    def __exit__(self, exc_type, exc_value, traceback):
        self.server.__exit__(exc_type, exc_value, traceback)

{% endhighlight %}

The `connect` method creates a mock server, and the `__enter__` method yields the mock server context manager. The `__exit__` method closes the mock server. The `__enter__` method also yields the `paramiko.SSHClient()` context manager, and returns the `ConnectorParamiko` object. This way, the `ConnectorParamiko` object can be used to mock the methods of `paramiko.SSHClient()`.

That's it. Forgive me if this was obvious, but I couldn't find a solution anywhere else. I hope this helps someone.
