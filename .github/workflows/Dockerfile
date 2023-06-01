
FROM hashicorp/terraform:1.4.6



WORKDIR /app


COPY . /app


EXPOSE 80

RUN terraform init
RUN terraform fmt -check
RUN terraform plan -input=false

CMD ["terraform", "apply"]
