.PHONY:	build push

TAG = 1.4

build:
	docker build -t drud/fluentd_logging_sidecar:$(TAG) .

push:
	docker push drud/fluentd_logging_sidecar:$(TAG)
