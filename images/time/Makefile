# 'Makefile'.
# Chris Shiels.


VERSION = latest


run:	build
	sudo docker run --rm -i -t -p 7002:7002 junio/time:$(VERSION)


build:	Dockerfile src/github.com/chris/time/time.go
	GOPATH=`pwd` go install -ldflags "-X main.version $(VERSION)" github.com/chris/time
	sudo docker build -t junio/time:$(VERSION) .
	touch $@
