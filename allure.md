# Allure - Getting Started

## Let's Chat

---

## How it's set up - Part 1 (Installs)

First, borrowed from [this gist](https://gist.github.com/npearce/6f3c7826c7499587f00957fee62f8ee9)

```
sudo amazon-linux-extras install docker
sudo service docker start
sudo usermod -a -G docker ec2-user
sudo chckconfig docker on
```

Then, I added `nginx`:

```
sudo amazon-linux-extras install nginx1
```

---

## How it's set up - Part 2 (Running docker)

Next, created a directory to sync with the s3 bucket
```
sudo su -
cd /home/ssm-user/
mkdir allure && cd allure
mkdir s3_bucket
```

and added a Dockerfile there to define the service
```
FROM frankescobar/allure-docker-service
ENV URL_PREFIX="/allure"
ENV CHECK_RESULTS_EVERY_SECONDS=NONE
ENV KEEP_HISTORY=1
ENV KEEP_HISTORY_LATEST=25
```

Then, we build and run the image
```
docker build -t allure:latest .
docker run -v "/${PWD}/s3_bucket:/app/allure-results" -p 5050:5050 allure:latest
```

Now we have allure up and waiting, we just need to get it access through the EC2's web port!

---

## How it's set up - Part 3 (Nginx)

First, we add a file in `/etc/nginx/default.d/allure.conf`
```
location /allure {
    rewrite /allure(.*) $1 break;
    proxy_pass http://localhost:5050;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Host $server_name;
}
```

This tells `nginx` to forward calls on the `/allure` path to our docker service on port 5050,
passing along the necessary headers.

We can now test this config:
`nginx -t`

Then start the service:
`service nginx start`

---

## How it's set up - Part 4 (gritty details)

One other thing I hit upon is that, for the docker service to be able to read/write our `s3_bucket` dir,
I had to modify the permissions:
```
chmod 777 s3_bucket
```

---
<!-- effect=fireworks -->

## The good news!

It works!

---

## How do we use it? = Part 1 (Repo Setup)

First, we make a few additions to an existing repo:
```
pip install allure-pytest
pip install git+ssh://git@github.com/nrccua/qa_tools.git
```

Then we need to add a project (uses `qa_tools` helper)
```
allure-add my-project-id
```

We can also view existing projects (`qa_tools` helpers):
```
allure-list
```

---

## How do we use it? -  Part 2 (Regular Usage)

And we can generate allure results (uses `allure-pytest`) from a run by specifying a target directory:
```
pytest --alluredir=allure_results my/test/path
```

Then upload the results afterwards (uses `qa_tools` helper):
```
allure-upload my-project-id allure_results
```

(This will upload the results and call to generate a new report)

Then view them on the [site](http://10.33.6.73/allure/projects/my-project-id/reports/latest/index.html)

Note: Allure stores only the latest *full results*,
and keeps a recent history of pass/fail from previous runs.

---

## Challenges

> The reward for a job well done is the opportunity to do more

- Currently, this is VPN-only
- Syncing data between the ec2 local directory and the S3 bucket is currently manual
  - S3 sync doesn't inherently go 2-way, so I have separate scripts to "push" and "pull"
  - We could put this on a cron schedule, or work a way to trigger it after every upload
  - There are 3rd party packages to mount the s3 bucket as a drive, but I'm not sure we want to go there yet
