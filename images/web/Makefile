# 'Makefile'.
# Chris Shiels.


VERSION = latest


run:	build
	sudo docker run --rm -i -t -p 7000:7000 junio/web:$(VERSION)


build:	Dockerfile src/github.com/chris/web/web.go
	GOPATH=`pwd` go install -ldflags "-X main.version=$(VERSION)" github.com/chris/web
	sudo docker build -t junio/web:$(VERSION) .
	touch $@
