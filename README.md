# nginx-proxy-minio-cluster
Nginx proxy MinIO cluster

## Nginx vhost proxy for MinIO
```
upstream minio_s3 {
   least_conn;
   server minio01:9000 max_fails=3 fail_timeout=30s;
   server minio02:9000 max_fails=3 fail_timeout=30s;
   server minio03:9000 max_fails=3 fail_timeout=30s;
   server minio04:9000 max_fails=3 fail_timeout=30s;
}

upstream minio_console {
   least_conn;
   server minio01:9001 max_fails=3 fail_timeout=30s;
   server minio02:9001 max_fails=3 fail_timeout=30s;
   server minio03:9001 max_fails=3 fail_timeout=30s;
   server minio04:9001 max_fails=3 fail_timeout=30s;
}

# MINIO API
server {
   listen       9000;
   listen  [::]:9000;
   server_name _;


   access_log /var/log/nginx/minio-api_access.log;
   error_log /var/log/nginx/minio-api_error.log;


   # Allow special characters in headers
   ignore_invalid_headers off;
   # Allow any size file to be uploaded.
   # Set to a value such as 1000m; to restrict file size to a specific value
   client_max_body_size 0;
   # Disable buffering
   proxy_buffering off;
   proxy_request_buffering off;

   location / {
      proxy_set_header Host $http_host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;

      proxy_connect_timeout 300;
      # Default is HTTP/1, keepalive is only enabled in HTTP/1.1
      proxy_http_version 1.1;
      proxy_set_header Connection "";
      chunked_transfer_encoding off;

      proxy_pass http://minio_s3; # This uses the upstream directive definition to load balance
   }

}


# MINIO CONSOLE UI
server {
   listen       9001;
   listen  [::]:9001;
   server_name _;


   access_log /var/log/nginx/minio-console_access.log;
   error_log /var/log/nginx/minio-console_error.log;

   # Allow special characters in headers
   ignore_invalid_headers off;
   # Allow any size file to be uploaded.
   # Set to a value such as 1000m; to restrict file size to a specific value
   client_max_body_size 0;
   # Disable buffering
   proxy_buffering off;
   proxy_request_buffering off;


   location / {
      proxy_set_header Host $http_host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_set_header X-NginX-Proxy true;

      # This is necessary to pass the correct IP to be hashed
      real_ip_header X-Real-IP;

      proxy_connect_timeout 300;

      # To support websockets in MinIO versions released after January 2023
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
      # Some environments may encounter CORS errors (Kubernetes + Nginx Ingress)
      # Uncomment the following line to set the Origin request to an empty string
      # proxy_set_header Origin '';

      chunked_transfer_encoding off;

      proxy_pass http://minio_console; # This uses the upstream directive definition to load balance
   }
}
```

## Nginx vhost proxy for MinIO custom URI
>[!NOTE]
> Reference: https://min.io/docs/minio/linux/integrations/setup-nginx-proxy-with-minio.html