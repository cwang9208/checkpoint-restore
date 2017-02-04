## CRIU
### Install CRIU
#### Dependencies

##### Protocol Buffers
###### Distribution Packages
The easiest way is to install distribution packages.
- Ubuntu
 -  sudo apt-get install --no-install-recommends git build-essential libprotobuf-dev libprotobuf-c0-dev protobuf-c-compiler protobuf-compiler python-protobuf libnl-3-dev libpth-dev pkg-config libcap-dev asciidoc libnet

#### Building CRIU From Source
##### Native Compilation
Simply run `make` in the CRIU source directory.

#### Installation
CRIU works perfectly even when run from the sources directory (with the "./criu" command), but if you want to have in standard paths run `make install`.  
You may need to install the following packages to generate docs in Debian-based OS's to avoid errors from install-man: 
- `asciidoc`
- `xmlto`

#### Checking That It Works
First thing to do is to run `criu check`. At the end it should say "Looks OK", if it doesn't the messages on the screen explain what functionality is missing.

### Simple TCP pair
The `TCP_REPAIR` socket option was added to the kernel 3.5 to help with C/R for TCP sockets. 

#### Compile and run simple TCP client and server
The below program is a simple echo server and a client, that pings server with increasing numbers once a second. In this howto we will dump and restore the client part.

```
#include <sys/socket.h>
#include <linux/types.h>
#include <sys/types.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <signal.h>

static int serve_new_conn(int sk)
{
	int rd, wr;
	char buf[1024];

	printf("New connection\n");

	while (1) {
		rd = read(sk, buf, sizeof(buf));
		if (!rd)
			break;

		if (rd < 0) {
			perror("Can't read socket");
			return 1;
		}

		wr = 0;
		while (wr < rd) {
			int w;

			w = write(sk, buf + wr, rd - wr);
			if (w <= 0) {
				perror("Can't write socket");
				return 1;
			}

			wr += w;
		}
	}

	printf("Done\n");
	return 0;
}

static int main_srv(int argc, char **argv)
{
	int sk, port, ret;
	struct sockaddr_in addr;

	/*
	 * Let kids die themselves
	 */

	signal(SIGCHLD, SIG_IGN);

	sk = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP);
	if (sk < 0) {
		perror("Can't create socket");
		return -1;
	}

	port = atoi(argv[1]);
	memset(&addr, 0, sizeof(addr));
	addr.sin_family = AF_INET;
	addr.sin_addr.s_addr = htonl(INADDR_ANY);
	addr.sin_port = htons(port);

	printf("Binding to port %d\n", port);

	ret = bind(sk, (struct sockaddr *)&addr, sizeof(addr));
	if (ret < 0) {
		perror("Can't bind socket");
		return -1;
	}

	ret = listen(sk, 16);
	if (ret < 0) {
		perror("Can't put sock to listen");
		return -1;
	}

	printf("Waiting for connections\n");
	while (1) {
		int ask, pid;

		ask = accept(sk, NULL, NULL);
		if (ask < 0) {
			perror("Can't accept new conn");
			return -1;
		}

		pid = fork();
		if (pid < 0) {
			perror("Can't fork");
			return -1;
		}

		if (pid > 0)
			close(ask);
		else {
			close(sk);
			ret = serve_new_conn(ask);
			exit(ret);
		}
	}
}

static int main_cl(int argc, char **argv)
{
	int sk, port, ret, val = 1, rval;
	struct sockaddr_in addr;

	sk = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP);
	if (sk < 0) {
		perror("Can't create socket");
		return -1;
	}

	port = atoi(argv[2]);
	printf("Connecting to %s:%d\n", argv[1], port);
	memset(&addr, 0, sizeof(addr));
	addr.sin_family = AF_INET;
	ret = inet_aton(argv[1], &addr.sin_addr);
	if (ret < 0) {
		perror("Can't convert addr");
		return -1;
	}
	addr.sin_port = htons(port);

	ret = connect(sk, (struct sockaddr *)&addr, sizeof(addr));
	if (ret < 0) {
		perror("Can't connect");
		return -1;
	}

	while (1) {
		write(sk, &val, sizeof(val));
		rval = -1;
		read(sk, &rval, sizeof(rval));
		printf("PP %d -> %d\n", val, rval);
		sleep(2);
		val++;
	}
}

int main(int argc, char **argv)
{
	if (argc == 2)
		return main_srv(argc, argv);
	else if (argc == 3)
		return main_cl(argc, argv);

	printf("Bad usage\n");
	return 1;
}
```

Compile it and run the server:
```
# ./tcp-howto <some-port>
```

On another terminal (for better output readability) run the client:
```
# ./tcp-howto 127.0.0.1 <some-port>
```

#### Try to dump the client
Create a directory for images (`img-dir/` below) and dump the client
```
# sudo criu dump -t <tcp-howto-client-pid> --images-dir img-dir/ -v 4 -o dump.log --shell-job --tcp-established
```
The state of the process(es) is saved to a few files:
```
$  ls img-dir/
core-15976.img  inetsk.img         pstree.img           tty.img
dump.log        inventory.img      reg-files.img        tty-info.img
fdinfo-2.img    mm-15976.img       sigacts-15976.img
fs-15976.img    pagemap-15976.img  stats-dump
ids-15976.img   pages-1.img        tcp-stream-acca.img
```
The `--tcp-established` option is a must, since client have active TCP connection and we should explicitly inform crtools about it.

#### Then restore the client
```
# sudo criu restore --images-dir img-dir/ -v 4 -o rst.log --shell-job --tcp-established
```

## LXC

```
sudo add-apt-repository ppa:ubuntu-lxc/daily
sudo apt-get update
sudo apt-get install lxc
lxc-create --version
  1.1.0
```

Create a new container named "u1" (if this container does not exist in /var/lib/lxc/u1)
```
sudo lxc-create -t ubuntu -n u1 -- -r trusty -a amd64
```
This command will take a few minutes to download all trusty (14.04) initial packages.

Start the server for the first time, so that we have "/var/lib/lxc/u1/config and /var/lib/lxc/u1/fstab".
```
sudo lxc-start -n u1
sudo lxc-stop -n u1
```
Config u1 fstab to share memory between host OS and container.
```
sudo echo "/dev/shm dev/shm none bind,create=dir" > /var/lib/lxc/u1/fstab

The first field describes the block special device or remote filesystem to be mounted.
The second field describes the mount point for the filesystem.
```

Append these lines to `/var/lib/lxc/u1/config`
```
lxc.network.ipv4 = 10.0.3.111/16
lxc.console = none
lxc.tty = 0
lxc.cgroup.devices.deny = c 5:1 rwm
lxc.rootfs = /var/lib/lxc/u1/rootfs
lxc.mount = /var/lib/lxc/u1/fstab
lxc.mount.auto = proc:rw sys:rw cgroup-full:rw
lxc.aa_profile = unconfined
```

And restart the container.
```
sudo lxc-start -n u1
```
Check the IP.
```
ssh ubuntu@10.0.3.111
 Enter password: "ubuntu"
```
This "ubuntu" user is already a sudoer. 

Then, you are free to install gcc, criu, and other packages in this container.

Add your own username into the lxc without asking password anymore.

For example:
- Generate a private/public key pair, put it into your ~/.ssh/.
- Rename the private key to be ~/.ssh/lxc_priv_key
- Append the public key to the u1 container's ~/.ssh/auth..._keys
- Then
```
ssh your_user_name@10.0.3.111 -i ~/.ssh/lxc_priv_key

-i identity_file
        Selects a file from which the identity (private key) for public key authentication is read.
```
Make sure you can login without entering password.

When you run sudo in the u1 container, avoid asking sudo password, append this line to /etc/sudoers
```
# Allow members of group sudo to execute any command
%sudo	ALL=(ALL:ALL) ALL

your_user_name ALL = NOPASSWD : ALL
```
Note: When multiple entries match for a user, they are applied in order. Where there are multiple matches, the last match is used (which is not necessarily the most specific match).
