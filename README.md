# LakeFS demo

This is a demo of a self-hosted instance of lakefs and its supporting architecture.

## To run it:

Ensure that you have a working installation of [helm](https://helm.sh/), [docker](https://www.docker.com/), and some local kebernetes engine like [minikube](https://minikube.sigs.k8s.io/docs/start/).

From `helm/` run `helm install lakefs-ecosystem lakefs-ecosystem`, this will install the following on your local kubernets cluster:

- postgresql + pv + pvc + svc
- minio + pv + pvc + svc
- lakefs + svc
- duckdb cluster + svc

Next you will need to port-forward your lakefs pod to be able to access it from your browser.

Run `kubectl get pods` and copy the pod named `lakefs-ecosystem-lakefs-instance-<random_string>`.
Next, `kubectl port-forward <paste pod name> 8000` to expose the port to your machine.
Now you can access lakefs from `localhost:8000`. In order to create a repository, you'll need to create a bucket in minio first.
Port-forwarding minio is a bit flaky for auth (after that it's fine), possibly due to websockets, but use this to port forward:
`while true; do kubectl port-forward pod/minio 9000 9090; done`
Now you can visit `localhost:9090`, credentials are minioadmin, minioadmin.
If logging in gets hung, ctrl-c the portforward, and it will automatically restart itself. Now try loggingin again.

Now create a basic bucket to use with lakefs. When creating a new reop in lakefs, reference this new bucket with
`s3://<bucket-name>`.

The duckdb bundle doesn't quite work right out of the box. If you'd like to use it to access / update the data in your mino, you'll need to follow these instructions.

- `kubectl exec -it <duckdb pod name> sh` to get into the pod

```
wget https://github.com/duckdb/duckdb/releases/download/v0.8.1/duckdb_cli-linux-amd64.zip
unzip duckdb_cli-linux-amd64.zip

./duckdb
INSTALL httpfs;
LOAD httpfs;

SET s3_url_style='path';
SET s3_region='us-east-1';
SET s3_use_ssl=false;
.changes on

SET s3_access_key_id='minioadmin';
SET s3_secret_access_key='minioadmin';
SET s3_endpoint='http://minio:9000';
```

Now you're free to perform sql on your data, for example:
`SELECT * FROM read_parquet('s3://<valid path to file in minio>');`

## Tools

### LakeFS

https://lakefs.io/
LakeFS is an open source project which provides version control for data in a git-like pattern, minimizing the learning needed by developers. It enables the write-audit-publish process to take place with complete audit logs, the addition of metadata, and the ability to audit in-browser. To avoid the common pitfall of data redundancy found in data lakes, LakeFS offers zero-copy import for S3, GCS, and Azure by copying pointers to the data in object storage rather than the data itself.
LakeFS uses a [postgres](https://www.postgresql.org/) database to store it's working data (branching data etc.).

### MinIO

https://min.io/
MinIO provides S3 compatible local storage to act as the data store lakefs is managing. To point this demo at an existing S3 instance, simply change the `blockstore` values the lakefs config found in `helm/lakefs-ecosystem/values.yaml` `lakefs-instance.lakefsConfig`.

To port-forward this, sometimes you need to restart it quickly so run `while true; do kubectl port-forward pod/minio 9000 9090; done` and if it gets hung up logging in, just ctrl-c the port forward and it'll restart.

### duckDB

https://duckdb.org/
DuckDB is an in-process. SQL OLAP database management system.

The bundle used in this repo was found [here](https://abcabhishek.substack.com/p/duckdb-bundle-on-kubernetes).
