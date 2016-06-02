# 'Makefile'.
# Chris Shiels.


VERSION = latest


run:	build
	sudo docker run --rm -i -t -p 8000:8000 junio/hello:$(VERSION)


build:	Dockerfile hello.js
	sudo docker build -t junio/date:$(VERSION) .
	touch $@
