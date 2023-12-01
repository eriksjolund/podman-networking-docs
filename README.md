# podman-networking-docs

This guide is about how to configure networking when using __rootless Podman__.

## Inbound TCP/UDP connections

### Overview

Listening TCP/UDP sockets

| method | source address preserved | native perfomance | support for binding to specific network device | minimum port number |
|-|-|-|-|-|
| [socket activation (systemd user service)](#socket-activation-systemd-user-service) | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | [ip_unprivileged_port_start](https://github.com/eriksjolund/podman-networking-docs#configure-ip_unprivileged_port_start) |
| [socket activation (systemd system service with User=)](#socket-activation-systemd-system-service-with-user) | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | 0 |
| pasta             | :heavy_check_mark: | | :heavy_check_mark: | [ip_unprivileged_port_start](https://github.com/eriksjolund/podman-networking-docs#configure-ip_unprivileged_port_start) |
| slirp4netns + port_handler=slirp4netns | :heavy_check_mark: | | | [ip_unprivileged_port_start](https://github.com/eriksjolund/podman-networking-docs#configure-ip_unprivileged_port_start) |
| slirp4netns + port_handler=rootlesskit | | | | [ip_unprivileged_port_start](https://github.com/eriksjolund/podman-networking-docs#configure-ip_unprivileged_port_start) |
| host | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | [ip_unprivileged_port_start](https://github.com/eriksjolund/podman-networking-docs#configure-ip_unprivileged_port_start) |

### Source address preserved

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

### Performance

| method | native perfomance |
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

### Support for binding to specific network device

| method | support for binding to specific network device |
|-|-|
| socket activation (systemd user service) | :heavy_check_mark: |
| socket activation (systemd system service) | :heavy_check_mark: |
| pasta | :heavy_check_mark: |
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
Network=pasta:-t,0.0.0.0%eth0/8080
PublishPorts=0.0.0.0:8080:8080
```

under the `[Container]` section in the container file.

For example the file _/home/test/.config/containers/systemd/nginx.container_ containing

   ```
   [Container]
   Image=ghcr.io/nginxinc/nginx-unprivileged:latest
   ContainerName=mynginx
   Network=pasta:-t,0.0.0.0%eth0/8080
   PublishPort=0.0.0.0:8080:8080
   
   [Install]
   WantedBy=default.target
   ```

If you want to publish an UDP port instead of a TCP port, replace `-t` with `-u` above.

-------------

</details>


### Configure `ip_unprivileged_port_start`

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

## Outbound TCP/UDP connections

### Outbound TCP/UDP connections to the internet

An example of an outbound TCP/UDP connection to the internet
is when a container downloads a file from a
web server on the internet.

| method | native perfomance |
|-|-|
| pasta | |
| slirp4netns | |
| host | :heavy_check_mark: |

### Outbound TCP/UDP connections to the host's localhost

An example of an outbound TCP/UDP connection to the host's localhost
is when a container downloads a file from a web server on the host that
listens on 127.0.0.1:80.

| method | outbound TCP/UDP connection to the host's localhost allowed by default |
|-|-|
| pasta | |
| slirp4netns | |
| host | :heavy_check_mark: |

Connecting to the host's localhost is not enabled by default for _pasta_ and _slirp4netns_ due to security reasons.
See network mode [`host`](#host) as to why access to the host's localhost is considered insecure.

To allow curl in a container to connect to a web server on the host that listens on 127.0.0.1:80,

for pasta add the option `--map-gw`

```
podman run --rm \
           --network=pasta:--map-gw \
           registry.fedoraproject.org/fedora curl localhost 10.0.2.2:80
```

and for slirp4netns add the option `slirp4netns:allow_host_loopback=true`

```
podman run --rm \
           --network=slirp4netns:allow_host_loopback=true \
	   registry.fedoraproject.org/fedora curl localhost 10.0.2.2:80
```

### Connecting to Unix socket on the host

| method | description |
|-|-|
| systemd directive `OpenFile=` | The executed command in the container inherits a file descriptor to an already opened file. |
| bind-mount, (--volume ./dir:/dir:Z ) | Standard way |

The systemd directive [`OpenFile=`](https://www.freedesktop.org/software/systemd/man/systemd.service.html#OpenFile=) was introduced in __systemd 253__ (released February 2023).

See also
https://github.com/eriksjolund/podman-OpenFile

## Valid method combinations

The methods

* pasta
* slirp4netns + port_handler=rootlesskit
* slirp4netns + port_handler=slirp4netns
* host

are mutually exclusive.

Socket activation can be combined with the other methods.

## Description of the different methods

### Socket activation (systemd user service)

This method can only be used for container images that has software that supports socket activation.

Socket activation of a systemd user service is set up by creating two files

* _~/.config/systemd/user/example.socket_

and either a Quadlet file

* _~/.config/containers/systemd/example.container_

or a service unit

* _~/.config/systemd/user/example.service_

See [Socket activation](https://github.com/containers/podman/blob/main/docs/tutorials/socket_activation.md#example-socket-activated-echo-server-container-in-a-systemd-service)

### Socket activation (systemd system service with `User=`)

:warning: Running Podman in a systemd system service with `User=` is not
yet supported (see feature request https://github.com/containers/podman/issues/12778)
but if you are willing to experiment (that is to patch and rebuild Podman), you might get this to work.

Although root privileges are required to create a system system service, note that rootless Podman is being used when `User=` is set under the `[Service]` section in the service unit file.

Socket activation of a systemd system service is set up by creating two files

* _/etc/systemd/system/example.socket_
* _/etc/systemd/system/example.service_

and running

```
systemctl daemon-reload
systemctl start example.socket
```
and possibly

```
systemctl enable example.socket
```
to start the socket automatically after a reboot.

See also https://github.com/containers/podman/issues/12778#issuecomment-1586255815
which describes a successful experiment to run Podman in a systemd system service
with `User=`. The experiment depends on a patched Podman. The suggested approach also needs to
add more security checks to be safe.

### Pasta

Pasta is similar to Slirp4netns. Pasta is generally the better choice because it is often faster and has more features than slirp4netns.

See the [`--network`](https://docs.podman.io/en/latest/markdown/podman-run.1.html#network-mode-net) option.
See also the pasta web page https://passt.top/

### Slirp4netns

slirp4netns is enabled by default if no `--network` option is provided to `podman run`.

The two port forwarding modes allowed with slirp4netns are described in 
https://news.ycombinator.com/item?id=33255771

See the [`--network`](https://docs.podman.io/en/latest/markdown/podman-run.1.html#network-mode-net) option.

### Host

:warning: Using `--network=host` is considered insecure.

Quote from [podman run man page](https://docs.podman.io/en/latest/markdown/podman-run.1.html#network-mode-net):
_"The host mode gives the container full access to local system services such as D-bus and is therefore considered insecure"._

See also the article [_[CVE-2020–15257] Don’t use --net=host . Don’t use spec.hostNetwork_](https://medium.com/nttlabs/dont-use-host-network-namespace-f548aeeef575) that explains why running containers in the host network namespace is insecure.