
FROM hashicorp/terraform:1.4.6


WORKDIR /app


COPY main.tf outputs.tf variables.tf versions.tf /app/


RUN apk add --no-cache curl


RUN terraform init
RUN terraform fmt -check
RUN terraform plan -input=false

CMD ["echo", "Docker container is running!"]
