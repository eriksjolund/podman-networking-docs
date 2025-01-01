# podman-networking-docs

This guide is about how to configure networking when using __rootless Podman__.

# Inbound TCP/UDP connections

## Overview

Listening TCP/UDP sockets

| method | source address preserved | native performance | support for binding to specific network device | minimum port number |
|-|-|-|-|-|
| [socket activation (systemd user service)](#socket-activation-systemd-user-service) | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | [ip_unprivileged_port_start](https://github.com/eriksjolund/podman-networking-docs#configure-ip_unprivileged_port_start) |
| [socket activation (systemd system service with User=)](#socket-activation-systemd-system-service-with-user) | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | 0 |
| pasta             | :heavy_check_mark: | | :heavy_check_mark: | [ip_unprivileged_port_start](https://github.com/eriksjolund/podman-networking-docs#configure-ip_unprivileged_port_start) |
| pasta + custom network | | | | [ip_unprivileged_port_start](https://github.com/eriksjolund/podman-networking-docs#configure-ip_unprivileged_port_start) |
| slirp4netns + port_handler=slirp4netns | :heavy_check_mark: | | | [ip_unprivileged_port_start](https://github.com/eriksjolund/podman-networking-docs#configure-ip_unprivileged_port_start) |
| slirp4netns + port_handler=rootlesskit | | | | [ip_unprivileged_port_start](https://github.com/eriksjolund/podman-networking-docs#configure-ip_unprivileged_port_start) |
| host | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | [ip_unprivileged_port_start](https://github.com/eriksjolund/podman-networking-docs#configure-ip_unprivileged_port_start) |

## Valid method combinations

The methods

* pasta
* pasta + custom network
* slirp4netns + port_handler=rootlesskit
* slirp4netns + port_handler=slirp4netns
* host

are mutually exclusive.

Socket activation can be combined with the other methods.

For example, it is possible to combine __socket activation__ with __pasta + custom network__ to get __source address preserved__ and __native speed__ communication to an HTTP reverse proxy that
is running on a custom network.

## Source address preserved

Example:

If the source address __is preserved__ in the incoming TCP connection,
then nginx is able to see the IP address of host2 (192.0.2.10)
where the curl request is run.

``` mermaid
flowchart LR
    curl-->nginx["nginx container"]
    subgraph host1
    nginx
    end
    subgraph host2 ["host2 ip=192.0.2.10"]
    curl
    end
```

nginx logs the HTTP request as coming from _192.0.2.10_
```
192.0.2.10 - - [15/Jun/2023:07:41:18 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.1.1" "-"
```

If the source address __is not preserved__, then nginx sees another
source address in the TCP connection.
For example, if the nginx container is run with __slirp4netns + port_handler=rootlesskit__

```
podman run --network=slirp4netns:port_handler=rootlesskit \
           --publish 8080:80 \
           --rm \
           ghcr.io/nginxinc/nginx:latest
```

nginx logs the HTTP request as coming from _10.0.2.2_
```
10.0.2.2 - - [15/Jun/2023:07:41:18 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.1.1" "-"
```

#### example: socket activation (systemd user service) - source address preserved

<details>
  <summary>Click me</summary>

-------------

This example uses two computers 
* _host1.example.com_ (for running the nginx web server)
* _host2.example.com_ (for running curl)

1. On _host1_ create user _test_
   ```
   sudo useradd test
   ```
2. Open an interactive shell session for the user _test_
   ```
   sudo machinectl shell test@
   ```
3. Create directories
   ```
   mkdir -p ~/.config/containers/systemd
   mkdir -p ~/.config/systemd/user
   ```
4. Create the file _/home/test/.config/containers/systemd/nginx.container_ containing
   ```
   [Container]
   Image=ghcr.io/nginxinc/nginx-unprivileged:latest
   ContainerName=mynginx
   Environment="NGINX=3;"
   
   [Install]
   WantedBy=default.target
   ```
5. Create the file _/home/test/.config/systemd/user/nginx.socket_ containing
   ```
   [Unit]
   Description=nginx socket
   
   [Socket]
   ListenStream=0.0.0.0:8080
   
   [Install]
   WantedBy=default.target
   ```
6. Reload the systemd user manager
   ```
   systemctl --user daemon-reload
   ```
7. Pull the container image
   ```
   podman pull ghcr.io/nginxinc/nginx-unprivileged:latest
   ```
8. Start the socket
   ```
   systemctl --user start nginx.socket
   ```
9. Test the nginx web server by accessing it from _host2_
   1. Log in to _host2_
   2. Run curl
       ```
       curl host1.example.com:8080
       ```
   3. Log out from _host2_
10. Check the logs in the container _mynginx_
    ```
    podman logs mynginx 2> /dev/null | grep "GET /"
    ```
    The output should look something like
    ```
    192.0.2.10 - - [15/Jun/2023:07:41:18 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.1.1" "-"
    ```
    nginx logged the source address of the TCP connection to be 192.0.2.10 which matches the IP address of _host2.example.com_. Conclusion: the source address was preserved.

__A side-note:__ If the feature request https://trac.nginx.org/nginx/ticket/237 gets implemented, the `Environment="NGINX=3;"` could be removed. This example makes use of the fact that "_nginx includes an undocumented, internal socket-passing mechanism_" quote from https://freedesktop.org/wiki/Software/systemd/DaemonSocketActivation/

-------------

</details>

#### example: pasta - source address preserved

<details>
  <summary>Click me</summary>

-------------

This example uses two computers 
* _host1.example.com_ (for running the nginx web server)
* _host2.example.com_ (for running curl)

1. On _host1_ create user _test_
   ```
   sudo useradd test
   ```
2. Open an interactive shell session for the user _test_
   ```
   sudo machinectl shell test@
   ```
3. Create directories
   ```
   mkdir -p ~/.config/containers/systemd
   ```
4. Create the file _/home/test/.config/containers/systemd/nginx.container_ containing
   ```
   [Container]
   Image=ghcr.io/nginxinc/nginx-unprivileged:latest
   ContainerName=mynginx
   Network=pasta
   PublishPort=0.0.0.0:8080:8080
   
   [Install]
   WantedBy=default.target
   ```
5. Reload the systemd user manager
   ```
   systemctl --user daemon-reload
   ```
6. Pull the container image
   ```
   podman pull ghcr.io/nginxinc/nginx-unprivileged:latest
   ```
7. Start the service
   ```
   systemctl --user start nginx.service
   ```
8. Test the nginx web server by accessing it from _host2_
   1. Log in to _host2_
   2. Run curl
       ```
       curl host1.example.com:8080
       ```
   3. Log out from _host2_
9. Check the logs in the container _mynginx_
   ```
   podman logs mynginx 2> /dev/null | grep "GET /"
   ```
   The output should look something like
   ```
   192.0.2.10 - - [15/Jun/2023:07:55:03 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.1.1" "-"
   ```
   nginx logged the source address of the TCP connection to be 192.0.2.10 which matches the IP address of _host2.example.com_. Conclusion: the source address __is preserved__.

-------------

</details>

#### example: pasta + custom network - source address not preserved

<details>
  <summary>Click me</summary>

-------------

Follow the same steps as

[example: pasta - source address preserved](#example-pasta---source-address-preserved)

but replace `Network=pasta` with `Network=mynet.network`. Create the network unit
file `mynet.network` that defines a custom network. (_mynet_ is an arbitrarily chosen name)

In other words, replace step 4 with

4. Create the file _/home/test/.config/containers/systemd/nginx.container_ containing
   ```
   [Container]
   Image=ghcr.io/nginxinc/nginx-unprivileged:latest
   ContainerName=mynginx
   Network=mynet.network
   PublishPort=0.0.0.0:8080:8080
   
   [Install]
   WantedBy=default.target
   ```
   Create the file _/home/test/.config/containers/systemd/mynet.network_ containing
   ```
   [Network]
   ```

At step 9 you will see that the source address __is not preserved__. Instead of 192.0.2.10 (IP address for _host1.example.com_),
nginx instead logs the IP address 10.89.0.2.

   ```
   podman logs mynginx 2> /dev/null | grep "GET /"
   ```
   The output should look something like
   ```
   10.89.0.2 - - [24/Jun/2024:07:10:59 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.6.0" "-"
   ```

-------------

</details>

#### example: pasta + custom network + socket activation + libsdsock - source address preserved

<details>
  <summary>Click me</summary>

Status: experimental. (The use of [__libsdsock__](https://github.com/ryancdotorg/libsdsock) in this example makes it experimental)

Some container images do not support _socket activation_.
For example this Containerfile defines a container image that has a web server that does not support _socket activation_.

   ```
   FROM docker.io/library/fedora
   RUN dnf -y install python3
   CMD ["python3","-m", "http.server", "8080", "--bind", "0.0.0.0"]
   ```

Such a container is typically configured with the `PublishPort` instruction in the container unit (for example `PublishPort=8080:8080`).
Unfortunately, the source address of a TCP connection that the web server sees is not the IP address of the client when using pasta and a custom network.
In other words, the _source address is not preserved_.

This shortcoming can be fixed by using the library [__libsdsock__](https://github.com/ryancdotorg/libsdsock).
If we create a socket unit and let the the web server have the library __libsdsock__ preloaded, the curl client source address is preserved.

Note, this is a rather experimental approach.

1. Create directory _dir_

2. Create the file _dir/Containerfile_ with the contents
   ```
   FROM docker.io/library/fedora as myweb
   RUN dnf -y install python3
   CMD ["python3","-m", "http.server", "8080","--bind", "0.0.0.0"]
   
   FROM docker.io/library/fedora as libsdsock-builder
   RUN dnf -y install gcc git make systemd-devel
   RUN git clone https://github.com/ryancdotorg/libsdsock.git
   RUN cd /libsdsock && make && make install

   FROM myweb
   RUN dnf -y install systemd-libs
   COPY --from=libsdsock-builder /usr/local/lib/libsdsock.so /usr/local/lib/libsdsock.so
   ```
   Note the difference to the original Containerfile. The file _/usr/local/lib/libsdsock.so_ and the
   RPM package _systemd-libs_ are installed.
3. Build the container image
   ```
   podman build -t myweb dir/
   ```
4. Create directories
   ```
   mkdir -p ~/.config/containers/systemd
   mkdir -p ~/.config/systemd/user
   ```
5. Create the file _~/.config/systemd/user/mynet.network_ with the contents
   ```
   [Network]
   Internal=true
   ```
6. Create the file _~/.config/systemd/user/myweb.socket_ with the contents
   ```
   [Socket]
   ListenStream=0.0.0.0:8080
   
   [Install]
   WantedBy=sockets.target
   ```
7. Create the file _~/.config/containers/systemd/myweb.container_ with the contents
   ```
   [Unit]
   Requires=myweb.socket
   After=myweb.socket

   [Container]
   Image=localhost/myweb
   Network=mynet.network
   Environment=LD_PRELOAD=/usr/local/lib/libsdsock.so
   Environment=LIBSDSOCK_MAP=tcp://0.0.0.0:8080=myweb.socket

   [Install]
   WantedBy=default.target
   ```
8. Reload the systemd user manager
   ```
   systemctl --user daemon-reload
   ```
9. Start the socket
   ```
   systemctl --user start myweb.socket
   ```
10. Fetch a web page with curl
    ```
    curl -s http://localhost:8080 | head -2
    ```
    The following output is printed
    ```
    <!DOCTYPE HTML>
    <html lang="en">
    ```
11. Run the command
    ```
    journalctl --user -xequ myweb.service | tail -1
    ```
    The following output is printed
    ```
    Aug 04 15:57:26 mycomputer systemd-myweb[12829]: 127.0.0.1 - - [04/Aug/2024 15:57:26] "GET / HTTP/1.1" 200 -
    ```

__result:__ The client source address is preserved.

</details>

#### example: slirp4netns + port_handler=slirp4netns - source address preserved

<details>
  <summary>Click me</summary>

-------------

Follow the same steps as

[example: pasta - source address preserved](#example-pasta---source-address-preserved)

but replace `Network=pasta` with `Network=slirp4netns:port_handler=slirp4netns`.

In other words, replace step 4 with

4. Create the file _/home/test/.config/containers/systemd/nginx.container_ containing
   ```
   [Container]
   Image=ghcr.io/nginxinc/nginx-unprivileged:latest
   ContainerName=mynginx
   Network=slirp4netns:port_handler=slirp4netns
   PublishPort=0.0.0.0:8080:8080
   
   [Install]
   WantedBy=default.target
   ```

-------------

</details>

#### example: slirp4netns + port_handler=rootlesskit - source address not preserved

<details>
  <summary>Click me</summary>

-------------

Follow the same steps as

[example: pasta - source address preserved](#example-pasta---source-address-preserved)

but replace `Network=pasta` with `Network=slirp4netns:port_handler=rootlesskit`.

In other words, replace step 4 with

4. Create the file _/home/test/.config/containers/systemd/nginx.container_ containing
   ```
   [Container]
   Image=ghcr.io/nginxinc/nginx-unprivileged:latest
   ContainerName=mynginx
   Network=slirp4netns:port_handler=rootlesskit
   PublishPort=0.0.0.0:8080:8080
   
   [Install]
   WantedBy=default.target
   ```

At step 9 you will see that the source address __is not preserved__. Instead of 192.0.2.10 (IP address for _host1.example.com_),
nginx instead logs the IP address 10.0.2.100.

   ```
   podman logs mynginx 2> /dev/null | grep "GET /"
   ```
   The output should look something like
   ```
   10.0.2.100 - - [15/Jun/2023:07:55:03 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.1.1" "-"
   ```

-------------

</details>

#### example: host - source address preserved

<details>
  <summary>Click me</summary>

-------------

Follow the same steps as

[example: pasta - source address preserved](#example-pasta---source-address-preserved)

but remove the line `PublishPort=0.0.0.0:8080:8080` and replace `Network=pasta` with `Network=host`.

In other words, replace step 4 with

4. Create the file _/home/test/.config/containers/systemd/nginx.container_ containing
   ```
   [Container]
   Image=ghcr.io/nginxinc/nginx-unprivileged:latest
   ContainerName=mynginx
   Network=host
   
   [Install]
   WantedBy=default.target
   ```

-------------

</details>

## Performance

| method | native performance |
|-|-|
| socket activation (systemd user service) | :heavy_check_mark: |
| socket activation (systemd system service) | :heavy_check_mark: |
| pasta | |
| slirp4netns + port_handler=slirp4netns | |
| slirp4netns + port_handler=rootlesskit | |
| host | :heavy_check_mark: |

Best performance has

* socket activation (systemd user service)
* socket activation (systemd system service)
* host

where there is no slowdown compared to running directly on the host.

The other methods ordered from fastest to slowest:

1. pasta
2. slirp4netns + port_handler=rootlesskit
3. slirp4netns + port_handler=slirp4netns

## Support for binding to specific network device

| method | support for binding to specific network device |
|-|-|
| socket activation (systemd user service) | :heavy_check_mark: |
| socket activation (systemd system service) | :heavy_check_mark: |
| pasta | :heavy_check_mark: |
| pasta + custom network | |
| slirp4netns + port_handler=slirp4netns | |
| slirp4netns + port_handler=rootlesskit | |
| host | :heavy_check_mark: |

examples

-------------

</details>

#### example: socket activation (systemd user service) - bind to specific network device

<details>
  <summary>Click me</summary>

-------------

Specify the network device to bind to with the systemd directive [BindToDevice](https://www.freedesktop.org/software/systemd/man/systemd.socket.html#BindToDevice=) in the socket unit file.

For example, to bind to the ethernet interface _eth0_, add the line
```
BindToDevice=eth0
```
The socket unit file could look like this 

```
[Unit]
Description=example socket

[Socket]
ListenStream=0.0.0.0:8080
BindToDevice=eth0

[Install]
WantedBy=default.target
```


-------------

</details>

#### example: pasta - bind to specific network device

<details>
  <summary>Click me</summary>

-------------

To publish the TCP port 8080 and bind the listening socket to the
ethernet interface _eth0_ use the configuration lines

```
Network=pasta:-t,0.0.0.0%%eth0/8080:8080
```

under the `[Container]` section in the container file.

For example the file _/home/test/.config/containers/systemd/nginx.container_ containing

   ```
   [Container]
   Image=ghcr.io/nginxinc/nginx-unprivileged:latest
   ContainerName=mynginx
   Network=pasta:-t,0.0.0.0%%eth0/8080:8080
   
   [Install]
   WantedBy=default.target
   ```

If you want to publish an UDP port instead of a TCP port, replace `-t` with `-u` above.

__Side note 1:__ The quadlet configuration directive `PublishPort=` is not used.
The port is in this example published by specifying the pasta `-t` option.

__Side note 2:__ Due to how quadlet/systemd parses the configuration line, a percentage character
needs to be escaped by prepending it with an extra percentage character.

The percentage character should not be escaped when the network option is provided
as a command-line option for `podman run`, for example `--network=pasta:-t,0.0.0.0%eth0/8080:8080`

-------------

</details>


## Configure `ip_unprivileged_port_start`

Read the current setting

```
$ cat /proc/sys/net/ipv4/ip_unprivileged_port_start
1024
```

To set a new value (for example 443), create
the file _/etc/sysctl.d/99-mysettings.conf_
with the contents:

```
net.ipv4.ip_unprivileged_port_start=443
```

and reload the configuration

```
sudo sysctl --system
```

The setting is system-wide so changing it impacts all users on the system.

Giving this privilege to all users on the computer might not be what you want
because often you already know which systemd service should be listening on a privileged port.
If the software supports _socket activation_, an alternative is to set up a
systemd system service with `User=`. For details, see the
section [_Socket activation (systemd system service with User=)_](#socket-activation-systemd-system-service-with-user)

## Outbound TCP/UDP connections

### Outbound TCP/UDP connections to the internet

An example of an outbound TCP/UDP connection to the internet
is when a container downloads a file from a
web server on the internet.

| method | native performance |
|-|-|
| pasta | |
| slirp4netns | |
| host | :heavy_check_mark: |

## Outbound TCP/UDP connections to the host's localhost

An example of an outbound TCP/UDP connection to the host's localhost
is when a container downloads a file from a web server on the host that
listens on 127.0.0.1:80.

| method | outbound TCP/UDP connection to the host's localhost | comment |
|-|-|-|
| pasta |:heavy_check_mark: | enable with one of the pasta options `--map-gw`, `--map-host-loopback` or `--tcp-ns` (`-T`) |
| slirp4netns |:heavy_check_mark: | enable with the slirp4netns option `allow_host_loopback=true` |
| host | :heavy_check_mark: | |

Connecting to the host's localhost is not enabled by default for _pasta_ and _slirp4netns_ due to security reasons.
See network mode [`host`](#host) as to why access to the host's localhost is considered insecure.

Scenario: allow curl in a container to connect to a web server on the host that listens on 127.0.0.1:8080

#### example: connect to host's localhost using slirp4netns

Add the slirp4netns option `allow_host_loopback=true`

```
podman run --rm \
           --network=slirp4netns:allow_host_loopback=true \
           registry.fedoraproject.org/fedora curl 10.0.2.2:8080
```

The IP address `10.0.2.2` is a special address used by slirp4netns.

#### example: connect to host's localhost using the pasta option `--map-gw`

Add the pasta option `--map-gw` and connect to `10.0.2.2:8080`

```
podman run --rm \
           --network=pasta:--map-gw \
           registry.fedoraproject.org/fedora curl 10.0.2.2:8080
```

The IP address `10.0.2.2` is a special address used by pasta.

#### example: connect to host's localhost using the pasta option `--map-host-loopback`

Add the pasta option `--map-host-loopback=11.11.11.11` and connect to _11.11.11.11:8080_.
The IP address _11.11.11.11_ was chosen arbitrarily.

```
podman run -ti --rm --network=pasta:--map-host-loopback=11.11.11.11 docker.io/library/fedora curl -s -4 11.11.11.11:8080
```

#### example: connect to host's localhost using the pasta option `--tcp-ns` (`-T`)

For better performance and security, pasta offers an alternative to using `--map-gw`.
The option `-T` (`--tcp-ns`) configures TCP port forwarding from the container network namespace to the init network namespace.

```
podman run --rm \
           --network=pasta:-T,8081:8080 \
           registry.fedoraproject.org/fedora curl 127.0.0.1:8081
```

(Instead of the port number 8081, it would also have been possible to specify the port number 8080)

For more information about how to use pasta to connect to a service running on the host, see [GitHub comment](https://github.com/containers/podman/issues/22653#issuecomment-2108922749).

## Outbound TCP/UDP connections to the host's main network interface (e.g eth0)

An example of an outbound TCP/UDP connection to the host's main network interface
is when a container downloads a file from a web server that
listens on the host's main network interface.

| method | outbound TCP/UDP connection to the host's main network interface | comment |
|-|-|-|
| pasta | :heavy_check_mark: | Connect to _host.containers.internal_ or _host.docker.internal_ or a hostname set with `--add-host=example.com:host-gateway`. Requires podman 5.3.0. For earlier Podman versions, try to set pasta option `--map-guest-addr` (see [Documentation relevant to older Podman versions](#documentation-relevant-to-older-podman-versions)) |
| slirp4netns | :heavy_check_mark: | |
| host | :heavy_check_mark: | |

To try this out, first start a web server that listens on the IP address of the host's main network interface.

<details>
  <summary>Start a web server that listens on host's main network interface: Click me</summary>

Check the IP address of the host

1. To show the IP address of the host's main network interface, run the command
   ```
   hostname -I
   ```
   The following output is printed
   ```
   192.168.10.108 192.168.122.1 192.168.39.1 fd25:c7f8:948a:0:912d:3900:d5c4:45ad
   ```
   __result:__  The IP address of the host's main network interface is 192.168.10.108. (The first listed IP address)

Start the web server

__Alternative 1__

Requirement: Python3 installed on the host

1. `sudo useradd user1`
3. `sudo machinectl shell --uid=user1`
4. `mkdir ~/emptydir`
5. `cd ~/emptydir`
6. Run the command
   ```
   python3 -m http.server 8080 --bind 192.168.10.108
   ```
   Note, the previously detected IP address of the host's main network interface was given as value to the __--bind__ option.

__Alternative 2__

Run a socket-activated nginx container with rootless podman.

Requirement: Podman installed on the host

1. `sudo useradd user1`
2. `sudo machinectl shell --uid=user1`
3. `mkdir -p ~/.config/systemd/user`
4. `mkdir -p ~/.config/containers/systemd`
5. Create the file _~/.config/systemd/user/nginx.socket_ containing
   ```
   [Unit]
   Description=Example
   
   [Socket]
   ListenStream=192.168.10.108:8080
   
   [Install]
   WantedBy=sockets.target
   ```
   Note, `192.168.10.108` is the previously detected IP address of the host's main network interface.
   It is used in the value for the directive `ListenStream`.
6. Create the file _~/.config/containers/systemd/nginx.container_ containing
   ```
   [Unit]
   Requires=nginx.socket
   After=nginx.socket
   
   [Container]
   Image=ghcr.io/nginxinc/nginx-unprivileged
   Environment=NGINX=3;
   Network=none
   Volume=%h/nginx_conf_d:/etc/nginx/conf.d:Z
   [Install]
   WantedBy=default.target
   ```
7. Create a directory that will be bind-mounted to _/etc/nginx/conf.d_ in the container
   ```
   $ mkdir $HOME/nginx_conf_d
   ```
8. Create the file _$HOME/nginx_conf_d/default.conf_ with the contents
   ```
   server {
       listen 192.168.10.108:8080;
       server_name example.com;
       location / {
	   root   /usr/share/nginx/html;
	   index  index.html index.htm;
       }
       error_page   500 502 503 504  /50x.html;
       location = /50x.html {
	   root   /usr/share/nginx/html;
       }
   }
   ```
   The file contents were created with the command
   ```
   podman run --rm ghcr.io/nginxinc/nginx-unprivileged /bin/bash -c 'cat /etc/nginx/conf.d/default.conf | grep -v \# | sed "s/listen\s\+8080;/listen 192.168.10.108:8080;/g" | sed "s/  localhost/ example.com/g" | sed /^[[:space:]]*$/d' > default.conf
   ```
   Note, `192.168.10.108` is the previously detected IP address of the host's main network interface.
   The IP address is used in the value for the nginx configuration directive `listen`.
9. Reload the systemd user manager
   ```
   systemctl --user daemon-reload
   ```
10. Enable linger for the current user (_user1_)
    ```
    loginctl enable-linger
    ```
11. Pull the container image
    ```
    podman pull ghcr.io/nginxinc/nginx-unprivileged
    ```
12. Start the socket
    ```
    systemctl --user start nginx.socket
    ```
11. `exit`

</details>

#### example: connect to host's main network interface using slirp4netns

This example shows that `--network slirp4netns` allows a container to connect to a port on the host's main network interface.

<details>
  <summary>Click me</summary>

Run curl to access the web server

1. `sudo useradd user2`
2. `sudo machinectl shell --uid=user2`
3. `podman pull docker.io/library/fedora`
4. Run the command
   ```
   podman run --rm --network slirp4netns docker.io/library/fedora curl -s -4 192.168.10.108:8080 | head -4
   ```
   The following output is printed
   ```
   <!DOCTYPE html>
   <html>
   <head>
   <title>Welcome to nginx!</title>
   ```
   Note, `192.168.10.108` is the previously detected IP address of the host's main network interface.
5. `exit`
6. `sudo machinectl shell --uid=user1`
7. Check the nginx logs
   ```
   journalctl --user -q -xeu nginx.service | tail -1
   ```
   The following output is printed
   ```
   Aug 19 18:22:00 fcos systemd-nginx[16328]: 192.168.10.108 - - [19/Aug/2024:18:22:00 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.6.0" "-"
   ```
  _192.168.10.108_ is the IP address of the host's main network interface.

</details>

#### example: connect to host's main network interface using pasta

This example shows that pasta allows a container to connect to a port on the host's main network interface
by connecting to _host.containers.internal_, _host.docker.internal_ or to a custom hostname that is set with `--add-host=example.com:host-gateway`.
The example requires Podman 5.3.0 or later.

<details>
  <summary>Click me</summary>

To connect to  _example.com_, add `--add-host=example.com:host-gateway`

```
$ podman run --rm --add-host=example.com:host-gateway fedora curl -4 -s example.com:8080 | head -4
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

The web page was downloaded from an nginx server that listens on the host's main network interface.

Using a custom network works too.

```
$ podman network create mynet
$ podman run --rm --network mynet --add-host=example.com:host-gateway fedora curl -4 -s example.com:8080 | head -4
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

Instead of setting a hostname with `--add-host=example.com:host-gateway` you could also connect to
_host.containers.internal_ or _host.docker.internal_. Podman adds those hostnames by default to
_/etc/hosts_ in the container.

```
$ podman run --rm fedora cat /etc/hosts | grep host.containers.internal
169.254.1.2	host.containers.internal host.docker.internal
```

```
$ podman run --rm --add-host=example.com:host-gateway fedora cat /etc/hosts | grep -E 'example.com|host.containers.internal'
169.254.1.2	example.com
169.254.1.2	host.containers.internal host.docker.internal
```

</details>

## Connecting to Unix socket on the host

| method | description |
|-|-|
| systemd directive `OpenFile=` | The executed command in the container inherits a file descriptor to an already opened file. |
| bind-mount, (--volume ./dir:/dir:Z ) | Standard way |

The systemd directive [`OpenFile=`](https://www.freedesktop.org/software/systemd/man/systemd.service.html#OpenFile=) was introduced in __systemd 253__ (released February 2023).

See also
https://github.com/eriksjolund/podman-OpenFile

# Description of the different methods

## Socket activation (systemd user service)

This method can only be used for container images that has software that supports socket activation.

Socket activation of a systemd user service is set up by creating two files

* _~/.config/systemd/user/example.socket_

and either a Quadlet file

* _~/.config/containers/systemd/example.container_

or a service unit

* _~/.config/systemd/user/example.service_

See [Socket activation](https://github.com/containers/podman/blob/main/docs/tutorials/socket_activation.md#example-socket-activated-echo-server-container-in-a-systemd-service)

### Add socket activation support by preloading libsdsock with LD_PRELOAD

Status: experimental

Is it possible to use _socket activation_ when the executable in the container does not support _socket activation_?

Yes, sometimes.
If the executable in the container is dynamically linked, it might be possible to preload the library [__libsdsock__](https://github.com/ryancdotorg/libsdsock)
to add support for _socket activation_.

The container unit needs to set the environment variables `LD_PRELOAD` and `LIBSDSOCK_MAP`

```
Environment=LD_PRELOAD=/usr/local/lib/libsdsock.so
Environment=LIBSDSOCK_MAP=tcp://0.0.0.0:8080=myweb.socket
```
The container needs to have the systemd libraries (RPM package: systemd-libs) and the file /usr/local/lib/libsdsock.so installed.

See [example: pasta + custom network + socket activation + libsdsock - source address preserved](#example-pasta--custom-network--socket-activation--libsdsock---source-address-preserved)

## Socket activation (systemd system service with `User=`)

Systemd system service (`User=`) and socket activation makes it possible for rootless Podman to use privileged ports.

For details of how to use socket-activated nginx, see for instance
Example 3, Example 4, Example 5, Example 6 in the repo https://github.com/eriksjolund/podman-nginx-socket-activation

:warning: How well this solution works is currently unknown. What are the pros and cons? Will it work for other software than nginx? More testing is needed.

There is a [Podman feature request](https://github.com/containers/podman/discussions/20573)
for adding Podman support for `User=` in systemd system services.
The feature request was moved into a GitHub discussion.

## Pasta

Pasta is enabled by default if no `--network` option is provided to `podman run`.
Pasta is generally the better choice because it is often faster and has more features than slirp4netns.

On RPM-based systems the executable pasta is in the _passt_ RPM package.

Show the RPM package for the executable `/usr/bin/pasta`
```
$ rpm -qf --queryformat "%{NAME}\n" /usr/bin/pasta
passt
```
The RPM package _passt-selinux_ contains the SELinux configuration for pasta.

To install pasta on Fedora run

```
$ sudo dnf install -y passt passt-selinux
```

See the [`--network`](https://docs.podman.io/en/latest/markdown/podman-run.1.html#network-mode-net) option.
See also the pasta web page https://passt.top/


### Show the default rootlessNetworkCmd

Pasta is the default rootlessNetworkCmd since Podman 5.0.0 (released March 2024).

To show the rootlessNetworkCmd that is configured to be used by default, run

```
podman info -f '{{.Host.RootlessNetworkCmd}}'
```

If __jq__ is installed on the computer, then the same result is produced with

```
podman info -f json | jq -r .host.rootlessNetworkCmd
```

If __podman info__ does not support the field `RootlessNetworkCmd`, then
it's possible to find out the information by running

```
podman run -d --rm -p 12345 docker.io/library/alpine sleep 300
```
and observing if the helper process is `pasta` or `slirp4netns`.

For details:

<details>
  <summary>Click me</summary>

1. Set the shell variable `user` to a username that is not in use.
   ```
   user=mytestuser
   ```
1. Create the new user
   ```
   sudo useradd $user
   ```
1. Open a shell for the new user
   ```
   sudo machinectl shell --uid $user
   ```
1. Verify that no pasta processes are running as the new user.
   ```
   pgrep -u $USER pasta -l
   ```
   The command should not list any processes.
1. Verify that no slirp4netns processes are running as the new user.
   ```
   pgrep -u $USER slirp4netns -l
   ```
   The command should not list any processes.
1. Run container
   ```
   podman run -d --rm -p 12345 docker.io/library/alpine sleep 300
   ```
   (12345 is just an arbitrary container port number)
1. Check if there are any pasta processes running as the new user.
   ```
   pgrep -u $USER pasta -l
   ```
   If the command lists any processes, then pasta is detected as being the default.
1. Check if there are any slirp4netns processes running as the new user.
   ```
   pgrep -u $USER slirp4netns -l
   ```
   If the command lists any processes, then slirp4netns is detected as being the default.
1. Exit the shell
   ```
   exit
   ```
1. Optional step: Delete the newly created user

</details>

### Publish container ports with pasta

#### example:  use `podman run` option `-p`

The __podman run__ option [`-p`](https://docs.podman.io/en/latest/markdown/podman-run.1.html#publish-p-ip-hostport-containerport-protocol) (`--publish`) publishes
a container's port, or a range of ports, to the host.

This example shows that if __podman run__ is given `-p 8080:80`, then podman starts _pasta_ with the argument  `-t 8080-8080:80-80` (which is equivalent to `-t 8080:80`)

<details>
  <summary>Click me</summary>

1. Run an nginx container and publish container port 80 to host port 8080
   ```
   podman run -p 8080:80 \
              -d \
              --rm \
              --name test \
              docker.io/library/nginx
   ```
2. Fetch a web page with curl
   ```
   curl -s http://localhost:8080 | head -4
   ```
   The command prints the following output
   ```
   <!DOCTYPE html>
   <html>
   <head>
   <title>Welcome to nginx!</title>
   ```
3. Check command-line arguments of the pasta process
   ```
   pgrep -l -a pasta
   ```
   The command prints the following output
   ```
   851253 /usr/bin/pasta --config-net -t 8080-8080:80-80 --dns-forward 169.254.0.1 -u none -T none -U none --no-map-gw --quiet --netns /run/user/1004/netns/netns-830a424a-0592-361f-556b-7bef910405cf
   ```
   __result__: pasta was started with the option `-t 8080-8080:80-80` which is equivalent with `-t 8080:80`
4. Remove container
   ```
   podman container rm -t0 -f test
   ```

</details>

#### example: use pasta option `-t` to publish a port

Although ports are usually published by providing the __podman run__ option  [`-p`](https://docs.podman.io/en/latest/markdown/podman-run.1.html#publish-p-ip-hostport-containerport-protocol) (`--publish`) , this example shows that passing `--network pasta:-t,8080:80` is roughly equivalent to passing `-p 8080:80`

<details>
  <summary>Click me</summary>

1. Run an nginx container and publish container port 80 to host port 8080
   ```
   podman run --network pasta:-t,8080:80 \
              --rm \
              -d \
              --name test \
              docker.io/library/nginx
   ```
2. Fetch a web page with curl
   ```
   curl -s http://localhost:8080 | head -4
   ```
   The command prints the following output
   ```
   <!DOCTYPE html>
   <html>
   <head>
   <title>Welcome to nginx!</title>
   ```
3. Check command-line arguments of the pasta process
   ```
   pgrep -l -a pasta
   ```
   The command prints the following output
   ```
   851253 /usr/bin/pasta --config-net -t 8088-8088:80-80 --dns-forward 169.254.0.1 -u none -T none -U none --no-map-gw --quiet --netns /run/user/1004/netns/netns-830a424a-0592-361f-556b-7bef910405cf
   ```
   __result__: pasta was started with the option `-t 8088-8088:80-80` which is equivalent with `-t 8088:80`
4. Remove container
   ```
   podman container rm -t0 -f test
   ```

</details>

#### example: use pasta option `-t auto` to let pasta detect listening sockets

Let pasta check once a second for new listening sockets (TCP or UDP) in the container and automatically publish them.
Use `--network=pasta:-t,auto`

<details>
  <summary>Click me</summary>

1. Create directory _dir_

1. Create the file _dir/Containerfile_ with the contents
   ```
   FROM docker.io/library/fedora
   RUN dnf -y install iproute nmap-ncat
   ```
1. Build the container image
   ```
   podman build -t ncat dir/
   ```
1. Run `nc -l 1234` in the container (but first wait 60 seconds)
   ```
   podman run --network=pasta:-t,auto \
              --rm \
              -d \
              --name test \
              localhost/ncat bash -c "sleep 60 && nc -l 1234 && sleep inf"
   ```
   The container starts listening on port 1234 after a delay of 60 seconds. The delay
   is added to demonstrate that pasta will detect that a listening socket is created while
   the container is running.
1. Check if the listening TCP port 1234 has been published on the host
   ```
   $ ss -tlnp "sport = 1234"
   State                   Recv-Q                  Send-Q                                   Local Address:Port                                    Peer Address:Port                  Process
   ```
   __result:__ No
1. Wait 60 seonds and check again if the listening TCP port 1234 has been published on the host
   ```
   $ ss -tlnp "sport = 1234"
   State                   Recv-Q                  Send-Q                                   Local Address:Port                                    Peer Address:Port                  Process
   LISTEN                  0                       128                                            0.0.0.0:1234                                         0.0.0.0:*                      users:(("pasta",pid=2644,fd=146))
   ```
   __result:__ Yes. After 60 seconds `nc` in the container started listening on TCP port 1234. Pasta detected this and published TCP port 1234 on the host.
1. Remove container
   ```
   podman container rm -t0 -f test
   ```

</details>

__Side note__: Pasta does not publish TCP ports below [ip_unprivileged_port_start](https://github.com/eriksjolund/podman-networking-docs#configure-ip_unprivileged_port_start).

### Pasta documentation links

GitHub comments:

* [GitHub issue](https://github.com/containers/podman/issues/23883) mentions that the performance of pasta can be improved by adding the option `-o mtu=65520`  to the __podman network create__ command.

* [GitHub comment](https://github.com/containers/podman/discussions/22943#discussioncomment-9795883) with a diagram of how pasta sets up custom networks.
  The diagram shows an example similar to this
  ```
  podman network create mynet1
  podman network create mynet2
  podman run --network mynet1 --name container1 ...
  podman run --network mynet1 --network mynet2 --name container2 ...
  podman run --network mynet2 --name container4 ...
  ```
* [GitHub comment](https://github.com/containers/podman/issues/19213#issuecomment-1979948655) Comparing the design of pasta and slirp4netns regarding the use of NAT

Talks:

* March 2023 [_passt & pasta: Modern unprivileged networking for containers and VMs_](https://www.youtube.com/watch?v=QMUEtEt1i3I) from conference _Everything Open_ Melbourne, Australia.
* June 2023 [_Root is less: container networks get in shape with pasta - DevConf.CZ_](https://devconfcz2023.sched.com/event/1MYld/root-is-less-container-networks-get-in-shape-with-pasta) video: [youtube](https://www.youtube.com/watch?v=tlxDmUPc4WY), slides: [pdf](https://static.sched.com/hosted_files/devconfcz2023/b5/pasta_devconf.pdf)
* June 2024 [_Podman networking deep dive - DevConf.CZ_](https://pretalx.com/devconf-cz-2024/talk/BVM77L/) video: [youtube](https://youtu.be/MCY6APZ4x3A?si=Pp64Vcpa8l3qmfm-&t=1180), slides: [pdf](https://pretalx.com/media/devconf-cz-2024/submissions/BVM77L/resources/Devconf.cz_podman_networking_rQ3aMf1.pdf)

## Slirp4netns

Slirp4netns is similar to Pasta but is slower and has less functionality.
Slirp4netns was the default rootlessNetworkCmd before Podman 5.0.0 (released March 2024).

The two port forwarding modes allowed with slirp4netns are described in 
https://news.ycombinator.com/item?id=33255771

See the [`--network`](https://docs.podman.io/en/latest/markdown/podman-run.1.html#network-mode-net) option.

## Host

:warning: Using `--network=host` is considered insecure.

Quote from [podman run man page](https://docs.podman.io/en/latest/markdown/podman-run.1.html#network-mode-net):
_"The host mode gives the container full access to local system services such as D-bus and is therefore considered insecure"._

See also the article [_[CVE-2020–15257] Don’t use --net=host . Don’t use spec.hostNetwork_](https://medium.com/nttlabs/dont-use-host-network-namespace-f548aeeef575) that explains why running containers in the host network namespace is insecure.

# Network backends

Check which network backend is in use

```
$ podman info --format {{.Host.NetworkBackend}}
netavark
```

## CNI

The network backend CNI ([Container Network Interface](https://www.cni.dev/)) was removed in Podman 5.0.0.
The reasons for replacing CNI with Netavark are described in the article
[_Podman 4.0's new network stack: What you need to know_](https://www.redhat.com/sysadmin/podman-new-network-stack).

## Netavark

_Netavark_ is the default network backend.

**Example** Create a network and run an nginx container

Create the network _mynet_

```
$ podman network create mynet
```

Start the container __docker.io/library/nginx__ and let it be connected to the network _mynet_

```
$ podman run -d -q --network mynet docker.io/library/nginx
19f812cfbb43c022529b84bb9914cda2b16e55ef09c0bc8e937afddfc803f812
```

Check the IP address

```
$ podman container inspect -l --format "{{(index .NetworkSettings.Networks \"mynet\").IPAddress}}"
10.89.0.2
```

Try to fetch a web page from nginx

```
$ curl --max-time 3 10.89.0.2
curl: (28) Connection timed out after 3000 milliseconds
```
__result:__ curl was not able to connect to the web server

Join the rootless network namespace used for netavark networking before running the curl command

```
$ podman unshare --rootless-netns curl --max-time 3 10.89.0.2 | head -4
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```
__result:__ curl fetched the web page.

# Capture network traffic

The pasta option __--pcap__ enables capturing of network traffic.

__Example__

Capture a curl request with pasta to the file _myfile.pcap_.
Use `tshark` to analyse the file _myfile.pcap_.

1. Fetch web page from http://podman.io with `curl`
   ```
   podman run \
      --rm \
      --network=pasta:--pcap,myfile.pcap \
      docker.io/library/fedora curl http://podman.io
   ```
   pasta is configured to capture network traffic to the file _myfile.pcap_


Build tshark container image


1. Create directory
   ```
   mkdir ctr
   ```
1. Create the file _ctr/Containerfile_ with the contents
   ```
   FROM docker.io/library/fedora
   RUN dnf install -y tshark && dnf clean all
   ```
1. Build container image _tshark_
   ```
   podman build -t tshark ctr/
   ```

Show HTTP host and HTTP method in HTTP requests

1. Use __tshark__ to analyse the file _myfile.pcap_.
   ```
   podman run \
      --rm
      -v ./myfile.pcap:/mnt/myfile.pcap:Z,ro \
      --user 65534:65534 \
      --userns keep-id:uid=65534,gid=65534 \
      localhost/tshark \
        tshark \
          -r /mnt/myfile.pcap \
          -T fields \
          -e http.host \
          -e http.request.method \
          -Y http | sort -u
   ```
   The command prints the following output
   ```
        
   podman.io    GET
   ```

Show the destination address of IP packets.

1. Use __tshark__ to analyse the file _myfile.pcap_.
   ```
   podman run \
      --rm
      -v ./myfile.pcap:/mnt/myfile.pcap:Z,ro \
      --user 65534:65534 \
      --userns keep-id:uid=65534,gid=65534 \
      localhost/tshark \
        tshark \
          -r /mnt/myfile.pcap \
          -T fields \
          -e ip.dst | sort -u
   ```
   The command prints the following output
   ```
   
   10.0.2.15
   169.254.0.1
   185.199.110.153
   ```
2. Look up DNS A record of _podman.io_
   ```
   host -t a podman.io
   ```
   The command prints the following output
   ```
   podman.io has address 185.199.110.153
   podman.io has address 185.199.111.153
   podman.io has address 185.199.108.153
   podman.io has address 185.199.109.153
   ```
   The IP address 185.199.110.153 is also
   seen in the __tshark__ output in step 1.

# HTTP reverse proxy

Use an HTTP reverse proxy that supports socket activation to get better support for preserved source IP address
in incoming connections.

| software | socket activation support | systemd notify support | comment |
| --       | --                        | --                     | --      |
| caddy    | :heavy_check_mark:        | :heavy_check_mark:     | Reloading the caddy configuration does not work (see https://github.com/caddyserver/caddy/issues/6631) |
| nginx    | :heavy_check_mark:        |                        | Although _socket activation_ works, it is not officially supported by nginx. See feature request https://trac.nginx.org/nginx/ticket/237. |
| traefik  | :heavy_check_mark:        | :heavy_check_mark:     | Traefik has issues during startup otherwise it works fine after a few seconds. When Traefik starts up Traefik might return HTTP response 404. Traefik sends systemd notify `READY=1` before traefik is ready. See https://github.com/traefik/traefik/issues/7347 |

See examples:

* https://github.com/eriksjolund/podman-caddy-socket-activation
* https://github.com/eriksjolund/podman-nginx-socket-activation
* https://github.com/eriksjolund/podman-traefik-socket-activation

# Troubleshooting

### systemd user service generated from quadlet fails after reboot. Error message `External interface not usable`

__Symptom__

Pasta is used. A systemd user service _example.service_ is generated from the podman container unit file _example.container_. Such a file is also called a quadlet file. After a reboot
the _example.service_ fails to start. The journal log contains an error message

```
Error: pasta failed with exit code 1:
External interface not usable
```

__Solution__

The container needs to start after the systemd system target _network-online.target_ has become active.

systemd does not support defining dependencies between _systemd system targets_ and _systemd user services_.

There is a GitHub feature request for adding the functionality:

* https://github.com/systemd/systemd/issues/3312

As a workaround create a _systemd user service_ that runs `sh -c 'until systemctl is-active network-online.target; do sleep 0.5; done'`

Alternative 1:

Use Podman 5.3.0 or later which includes the file _/usr/lib/systemd/user/podman-user-wait-network-online.service_.

```
$ grep ExecStart= /usr/lib/systemd/user/podman-user-wait-network-online.service
ExecStart=sh -c 'until systemctl is-active network-online.target; do sleep 0.5; done'
```

Dependencies for that service is added by default by the quadlet generator (`/usr/lib/systemd/user-generators/podman-user-generator`)
when it generates systemd user services from user container units.

Alternative 2:

For older Podman versions follow these steps:

1. Create directory
   ```
   mkdir -p ~/.config/systemd/user/
   ```
1. Create the file _~/.config/systemd/user/podman-user-wait-network-online.service_
   ```
   curl -o ~/.config/systemd/user/podman-user-wait-network-online.service \
     -Ls https://raw.githubusercontent.com/containers/podman/refs/heads/main/contrib/systemd/user/podman-user-wait-network-online.service
   ```
2. Add the following lines to the existing file _~/.config/containers/systemd/example.container_ under the `[Unit]` section
   ```
   Wants=podman-user-wait-network-online.service
   After=podman-user-wait-network-online.service
   ```
3. Reload the systemd user manager
   ```
   systemctl --user daemon-reload
   ```
4. Enable _podman-user-wait-network-online.service_
   ```
   systemctl --user enable podman-user-wait-network-online.service
   ```

The systemd user service _podman-user-wait-network-online.service_ will be in the _activating_ state until the systemd system service _network-online.target_ is active.

To show the current state of the systemd user service _podman-user-wait-network-online.service_, run the command

```
systemctl --user show -P ActiveState podman-user-wait-network-online.service
```

To show the current state of the systemd system target _network-online.target_, run the command

```
systemctl show -P ActiveState network-online.target
```

The output should usually be one of

* `active`
* `activating`

### Laptop intermittent network connectivity issues. Error message `External interface not usable`

__Symptom__

podman / pasta is currently not prepared for the situation when running containers while traveling with a laptop.
A wireless network might not always be available while traveling. The network might come and go which
causes problems like the error message `External interface not usable`

```
$ podman run --rm alpine echo hello world
Error: pasta failed with exit code 1:
External interface not usable
```
See also [GitHub discussion thread](https://github.com/containers/podman/discussions/22737)

__Solution 1__

If network access is not needed, add `--network none`

```
$ podman run --rm --network none alpine echo hello world
hello world
```

__Solution 2__

If network access is needed, add `--network slirp4netns`

__Side note__: Using `--network host` should also work but it is not recommended due to security reasons.

# Documentation relevant to older Podman versions

Documentation relevant to Podman 5.2.2 and earlier versions

<details>
  <summary>Click me</summary>

### About the pasta option __--map-guest-addr__

Podman 5.3.0 or later sets the pasta option __--map-guest-addr__ by default.

If you are runnning an earlier Podman version, you could try to enable it yourself.

Requirement: passt-0^20240821.g1d6142f or newer. That version was released 21 August 2024.

In the example Podman 5.2.1 is used. Earlier Podman versions might work too.

The example shows that `--network=pasta:--map-guest-addr=11.11.11.11` allows a container to connect
to a port on the host's main network interface by connecting to _11.11.11.11_.
The IP address _11.11.11.11_ was chosen arbitrarily.

```
podman run --rm --network=pasta:--map-guest-addr=11.11.11.11 docker.io/library/fedora curl -s -4 11.11.11.11:8080 | head -4
```
The following output is printed
```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

The web page was downloaded from an nginx server that is listening on TCP port 8080 on the host's main network interface.
The IP address _11.11.11.11_ was chosen arbitrarily in the example.

If you want to use a specific hostname such as _example.com_, run the command
```
podman run --rm --network=pasta:--map-guest-addr=11.11.11.11 --add-host example.com:11.11.11.11 docker.io/library/fedora curl -s -4 example.com:8080 | head -4
```
The following output is printed
```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

#### example: connect to host's main network interface using pasta and custom network

This example shows that setting
```
pasta_options = ["--map-guest-addr","11.11.11.11"]
```
in the file _containers.conf_ allows containers
in a custom network to connect to the host's main network interface by
connecting to _11.11.11.11_. The IP address _11.11.11.11_ was chosen arbitrarily.

Requirement: passt-0^20240821.g1d6142f or newer. That version was released 21 August 2024.
In the example Podman 5.2.1 is used. Earlier Podman versions might work too.

1. Create directory
   ```
   mkdir -p ~/.config/containers
   ```
2. If the file _~/.config/containers/containers.conf_ does not exist, create the file with the command
   ```
   cp /usr/share/containers/containers.conf ~/.config/containers/
   ```
3. Check the current `pasta_options` setting
   ```
   grep pasta_options ~/.config/containers/containers.conf
   ```
   The following output is printed
   ```
   #pasta_options = []
   ```
4. Edit the file _~/.config/containers/containers.conf_ and
   replace the line
   ```
   #pasta_options = []
   ```
   with
   ```
   pasta_options = ["--map-guest-addr","11.11.11.11"]
   ```
   The IP address _11.11.11.11_ was chosen arbitrarily. Remember the IP address because it can
   be used to for connecting to ports listening on the host's main network interface.
5. Create a custom network
   ```
   podman network create mynet
   ```
6. Run `curl` to download a web page from a web server listening on the host's main network interface
   ```
   podman run --rm --network=mynet docker.io/library/fedora curl -s -4 11.11.11.11:8080 | head -4
   ```
   The following output is printed
   ```
   <!DOCTYPE html>
   <html>
   <head>
   <title>Welcome to nginx!</title>
   ```

</details>
