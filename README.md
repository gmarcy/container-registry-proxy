## TL,DR

A caching proxy for container images; allows centralised management of (multiple) registries and their authentication; caches images from *any* registry.
Caches the potentially huge blob/layer requests (for bandwidth/time savings), and optionally caches manifest requests ("pulls") to avoid rate-limiting.

### Avoiding Container Registry Pull Rate Limits with Caching

This proxy can be configured with the env var `ENABLE_MANIFEST_CACHE=true` which provides 
configurable caching of the manifest requests that may be throttled. You can then fine-tune other parameters to your needs.
Together with the possibility to centrally inject authentication (since 0.3x), this is probably one of the best ways to bring relief to your distressed cluster, while at the same time saving lots of bandwidth and time. 

Note: enabling manifest caching, in its default config, effectively makes some tags **immutable**. Use with care. The configuration ENVs are explained in the [Dockerfile](./Dockerfile), relevant parts included below.

```dockerfile
# Manifest caching tiers. Disabled by default, to mimick 0.4/0.5 behaviour.
# Setting it to true enables the processing of the ENVs below.
# Once enabled, it is valid for all registries.
# The envs *_REGEX represent a regex fragment, check entrypoint.sh to understand how they're used (nginx ~ location, PCRE syntax).
ENV ENABLE_MANIFEST_CACHE="false"

# 'Primary' tier defaults to 10m cache for frequently used/abused tags.
# - People publishing to production via :latest (argh) will want to include that in the regex
# - Heavy pullers who are being ratelimited but don't mind getting outdated manifests should (also) increase the cache time here
ENV MANIFEST_CACHE_PRIMARY_REGEX="(stable|nightly|production|test)"
ENV MANIFEST_CACHE_PRIMARY_TIME="10m"

# 'Secondary' tier defaults any tag that has 3 digits or dots, in the hopes of matching most explicitly-versioned tags.
# It caches for 60d, which is also the cache time for the large binary blobs to which the manifests refer.
# That makes them effectively immutable. Make sure you're not affected; tighten this regex or widen the primary tier.
ENV MANIFEST_CACHE_SECONDARY_REGEX="(.*)(\d|\.)+(.*)(\d|\.)+(.*)(\d|\.)+"
ENV MANIFEST_CACHE_SECONDARY_TIME="60d"

# The default cache duration for manifests that don't match either the primary or secondary tiers above.
# In the default config, :latest and other frequently-used tags will get this value.
ENV MANIFEST_CACHE_DEFAULT_TIME="1h"
```


## What?

Essentially, it's a [man in the middle](https://en.wikipedia.org/wiki/Man-in-the-middle_attack): an intercepting proxy based on `nginx`, to which all container image traffic is directed using the `HTTPS_PROXY` mechanism and injected CA root certificates. 

The main feature is container image layer/image caching, including layers served from S3, Google Storage, etc. 

As a bonus it allows for centralized management of container registry credentials, which can in itself be the main feature, eg in Kubernetes environments.

You configure the clients (_err... Kubernetes Nodes?_) once, and then all configuration is done on the proxy -- 
for this to work it requires inserting a root CA certificate into system trusted root certs.

## Hosted on GitHub Container Registry (ghcr.io)

- GitHub image is at `ghcr.io/gmarcy/container-registry-proxy:<version>`

## Usage (running the Proxy server)

- Run the proxy on a host close (network-wise: high bandwidth, same-VPC, etc) to the clients
- Expose port 3128 to the network
- Map volume `/registry_proxy_mirror_cache` for up to `CACHE_MAX_SIZE` (32gb by default) of cached images across all cached registries
- Map volume `/ca`, the proxy will store the CA certificate here across restarts. **Important** this is security sensitive.
- Env `ALLOW_PUSH` : This bypasses the proxy when pushing, default to false - if kept to false, pushing will not work.
- Env `CACHE_MAX_SIZE` (default `32g`): set the max size to be used for caching local container image layers. Use [Nginx sizes](http://nginx.org/en/docs/syntax.html).
- Env `ENABLE_MANIFEST_CACHE`, see the section on pull rate limiting.
- Env `REGISTRIES`: space separated list of registries to cache.
- Env `AUTH_REGISTRIES`: space separated list of `hostname:username:password` authentication info.
  - `hostname`s listed here should be listed in the REGISTRIES environment as well, so they can be intercepted.
- Env `AUTH_REGISTRIES_DELIMITER` to change the separator between authentication info. By default, a space: "` `". If you use keys that contain spaces (as with Google Cloud Registry), you should update this variable, e.g. setting it to `AUTH_REGISTRIES_DELIMITER=";;;"`. In that case, `AUTH_REGISTRIES` could contain something like `registry1.com:user1:pass1;;;registry2.com:user2:pass2`.
- Env `AUTH_REGISTRY_DELIMITER` to change the separator between authentication info *parts*. By default, a colon: "`:`". If you use keys that contain single colons, you should update this variable, e.g. setting it to `AUTH_REGISTRIES_DELIMITER=":::"`. In that case, `AUTH_REGISTRIES` could contain something like `registry1.com:::user1:::pass1 registry2.com:::user2:::pass2`.
- Env `PROXY_REQUEST_BUFFERING`: If push is allowed, buffering requests can cause issues on slow upstreams.
If you have trouble pushing, set this to `false` first, then fix remainig timeouts.
Default is `true` to not change default behavior.
ENV PROXY_REQUEST_BUFFERING="true"
- Timeouts ENVS - all of them can pe specified to control different timeouts, and if not set, the defaults will be the ones from `Dockerfile`. The directives will be added into `http` block.:
  - SEND_TIMEOUT : see [send_timeout](http://nginx.org/en/docs/http/ngx_http_core_module.html#send_timeout)
  - CLIENT_BODY_TIMEOUT : see [client_body_timeout](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_body_timeout)
  - CLIENT_HEADER_TIMEOUT : see [client_header_timeout](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_header_timeout)
  - KEEPALIVE_TIMEOUT : see [keepalive_timeout](http://nginx.org/en/docs/http/ngx_http_core_module.html#keepalive_timeout
  - PROXY_READ_TIMEOUT : see [proxy_read_timeout](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_read_timeout)
  - PROXY_CONNECT_TIMEOUT : see [proxy_connect_timeout](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_connect_timeout)
  - PROXY_SEND_TIMEOUT : see [proxy_send_timeout](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_send_timeout)
  - PROXY_CONNECT_READ_TIMEOUT : see [proxy_connect_read_timeout](https://github.com/chobits/ngx_http_proxy_connect_module#proxy_connect_read_timeout)
  - PROXY_CONNECT_CONNECT_TIMEOUT : see [proxy_connect_connect_timeout](https://github.com/chobits/ngx_http_proxy_connect_module#proxy_connect_connect_timeout)
  - PROXY_CONNECT_SEND_TIMEOUT : see [proxy_connect_send_timeout](https://github.com/chobits/ngx_http_proxy_connect_module#proxy_connect_send_timeout))


### Simple (no auth, all cache)
```bash
podman run --rm --name container_registry_proxy -it \
       -p 0.0.0.0:3128:3128 -e ENABLE_MANIFEST_CACHE=true \
       -v $(pwd)/registry_proxy_mirror_cache:/registry_proxy_mirror_cache \
       -v $(pwd)/registry_proxy_mirror_certs:/ca \
       gmarcy/container-registry-proxy:latest
```

### Container Registry auth

```bash
podman run --rm --name container_registry_proxy -it \
       -p 0.0.0.0:3128:3128 -e ENABLE_MANIFEST_CACHE=true \
       -v $(pwd)/registry_proxy_mirror_cache:/registry_proxy_mirror_cache \
       -v $(pwd)/registry_proxy_mirror_certs:/ca \
       -e REGISTRIES="k8s.gcr.io gcr.io quay.io your.own.registry another.public.registry" \
       -e AUTH_REGISTRIES="quay.io:quay_username:quay_password your.own.registry:username:password" \
       gmarcy/container-registry-proxy:latest
```

### Simple registries auth (HTTP Basic auth)

For regular registry auth (HTTP Basic), the `hostname` should be the registry itself... unless your registry uses a different auth server.

See the example above, adapt the `your.own.registry` parts (in both ENVs).

This should work for quay.io also, but I have no way to test.

### GitLab auth

GitLab may use a different/separate domain to handle the authentication procedure.

If you run GitLab on `git.example.com` and its registry on `reg.example.com`, you need to include both in `REGISTRIES` and use the primary domain for `AUTH_REGISTRIES`.

For GitLab.com itself the authentication domain should be `gitlab.com`.

```bash
podman run  --rm --name container_registry_proxy -it \
       -p 0.0.0.0:3128:3128 -e ENABLE_MANIFEST_CACHE=true \
       -v $(pwd)/registry_proxy_mirror_cache:/registry_proxy_mirror_cache \
       -v $(pwd)/registry_proxy_mirror_certs:/ca \
       -e REGISTRIES="reg.example.com git.example.com" \
       -e AUTH_REGISTRIES="git.example.com:USER:PASSWORD" \
       gmarcy/container-registry-proxy:latest
```

### Google Container Registry (GCR) auth

For Google Container Registry (GCR), username should be `_json_key` and the password should be the contents of the service account JSON. 
Check out [GCR docs](https://cloud.google.com/container-registry/docs/advanced-authentication#json_key_file). 

The service account key is in JSON format, it contains spaces ("` `") and colons ("`:`"). 

To be able to use GCR you should set `AUTH_REGISTRIES_DELIMITER` to something different than space (e.g. `AUTH_REGISTRIES_DELIMITER=";;;"`) and `AUTH_REGISTRY_DELIMITER` to something different than a single colon (e.g. `AUTH_REGISTRY_DELIMITER=":::"`).

Example with GCR using credentials from a service account from a key file `servicekey.json`:

```bash
podman run --rm --name container_registry_proxy -it \
       -p 0.0.0.0:3128:3128 -e ENABLE_MANIFEST_CACHE=true \
       -v $(pwd)/registry_proxy_mirror_cache:/registry_proxy_mirror_cache \
       -v $(pwd)/registry_proxy_mirror_certs:/ca \
       -e REGISTRIES="k8s.gcr.io gcr.io quay.io your.own.registry another.public.registry" \
       -e AUTH_REGISTRIES_DELIMITER=";;;" \
       -e AUTH_REGISTRY_DELIMITER=":::" \
       -e AUTH_REGISTRIES="gcr.io:::_json_key:::$(cat servicekey.json);;;quay.io:::quay_username:::quay_password" \
       gmarcy/container-registry-proxy:latest
```

### Kind Cluster

[Kind](https://github.com/kubernetes-sigs/kind/) is a tool for running local Kubernetes clusters using container “nodes”.

Because cluster nodes are containers, container-registry-proxy needs to be in the same network.

Example joining the _kind_ network and using hostname _container-registry-proxy_ as hostname :

```bash
podman run --rm --name container_registry_proxy -it \
       --net kind --hostname container-registry-proxy \
       -p 0.0.0.0:3128:3128 -e ENABLE_MANIFEST_CACHE=true \
       -v $(pwd)/registry_proxy_mirror_cache:/registry_proxy_mirror_cache \
       -v $(pwd)/registry_proxy_mirror_certs:/ca \
       gmarcy/container-registry-proxy:latest
```

Now deploy your Kind cluster and then automatically configure the nodes with the following script :

```bash
#!/bin/sh
KIND_NAME=${1-kind}
SETUP_URL=http://container-registry-proxy:3128/setup/systemd
pids=""
for NODE in $(kind get nodes --name "$KIND_NAME"); do
  podman exec "$NODE" sh -c "\
      curl $SETUP_URL \
      | sed s/podman\.service/containerd\.service/g \
      | sed '/Environment/ s/$/ \"NO_PROXY=127.0.0.0\/8,10.0.0.0\/8,172.16.0.0\/12,192.168.0.0\/16\"/' \
      | bash" & pids="$pids $!" # Configure every node in background
done
wait $pids # Wait for all configurations to end
```

## Configuring the clients / Kubernetes nodes / Linux clients

Let's say you setup the proxy on host `192.168.66.72`, you can then `curl http://192.168.66.72:3128/ca.crt` and get the proxy CA certificate.

On each host that is to use the cache:

- [Configure registry proxy](https://docs.docker.com/config/daemon/systemd/#httphttps-proxy) pointing to the caching server
- Add the caching server CA certificate to the list of system trusted roots.
- Restart `dockerd`

Do it all at once, tested on Ubuntu Xenial, Bionic, and Focal, all systemd based:

```bash
# Add environment vars pointing Podman to use the proxy
mkdir -p /usr/lib/systemd/system/docker.service.d
cat << EOD > /usr/lib/systemd/system/podman.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=http://192.168.66.72:3128/"
Environment="HTTPS_PROXY=http://192.168.66.72:3128/"
EOD

### UBUNTU
# Get the CA certificate from the proxy and make it a trusted root.
curl http://192.168.66.72:3128/ca.crt > /usr/share/ca-certificates/container_registry_proxy.crt
echo "container_registry_proxy.crt" >> /etc/ca-certificates.conf
update-ca-certificates --fresh
###

### CENTOS
# Get the CA certificate from the proxy and make it a trusted root.
curl http://192.168.66.72:3128/ca.crt > /etc/pki/ca-trust/source/anchors/container_registry_proxy.crt
update-ca-trust
###

# Reload systemd
systemctl daemon-reload

# Restart dockerd
systemctl restart podman.service
```

## Testing

Clear `podman.service` of everything not currently running: `podman system prune -a -f` *beware*

Then do, for example, `podman pull k8s.gcr.io/kube-proxy-amd64:v1.10.4` and watch the logs on the caching proxy, it should list a lot of MISSes.

Then, clean again, and pull again. You should see HITs! Success.

Do the same for `podman pull ubuntu` and rejoice.

Test your own registry caching and authentication the same way; you don't need `podman login`, or `.docker/config.json` anymore.

## Gotchas

- If you authenticate to a private registry and pull through the proxy, those images will be served to any client that can reach the proxy, even without authentication. *beware*
- Repeat, **this will make your private images very public if you're not careful**.
- ~~**Currently you cannot push images while using the proxy** which is a shame. PRs welcome.~~ **SEE `ALLOW_PUSH` ENV FROM USAGE SECTION.**
- Setting this on Linux is relatively easy.
