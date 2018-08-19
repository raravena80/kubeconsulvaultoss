Fluentd log shipper sidecar based on [fluentd-sidecar-gcp](https://github.com/kubernetes/contrib/blob/master/logging/fluentd-sidecar-gcp)

This image is changed to allow shipping logs with alternate formats
if $FILES_FORMAT is set. 

For example if environment var FILES_FORMAT=json then the `format json` line will be added to the config for each file (instead of the default `format none`
