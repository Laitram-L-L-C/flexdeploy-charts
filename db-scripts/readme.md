after you install FD, you can deploy a k8sutils pod, upload this , unzip it , enter the dir and run 

```
PGPASSWORD='somepassword' psql \
  -h chris-postgres.cpiersfclgnd.us-west-2.rds.amazonaws.com \
  -p 5432 \
  -U fd_admin \
  -d flexdeploy_db \
  -v ON_ERROR_STOP=1 \
  -f CreateFlexDeploySchemas.sql
```