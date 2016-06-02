# 'Dockerfile'.
# Chris Shiels.


from centos:7

expose 7000

env PORT=7000
env DATEENDPOINT=_date._tcp.date.default.svc.cluster.local
env TIMEENDPOINT=_time._tcp.time.default.svc.cluster.local

cmd GODEBUG=netdns=cgo /web -p $PORT -dateendpoint $DATEENDPOINT -timeendpoint $TIMEENDPOINT

copy bin/web /
