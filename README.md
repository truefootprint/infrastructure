## Infrastructure

This repository contains a [Packer](https://www.packer.io/) template that
builds an
[Amazon Machine Image](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html)
that can be run on EC2. The image contains running instances of the
[field-backend](https://github.com/truefootprint/field-backend) and
[tracker-backend](https://github.com/truefootprint/tracker-backend) Rails apps
as well as an Nginx web server, Puma service workers, a Postgres database and
SSH keys for logging onto the server.

### How to use

If you are using [aws-vault](https://github.com/99designs/aws-vault) to
securely store you AWS access keys, you can run the following script to launch
Packer and build an image:

```sh
$ ./bin/build-ec2-image
```

This communicates with AWS to launch an EC2 instance and run the installation
and setup scripts for the applications. Once complete, the instance saves a
snapshot image and shuts down. This image may then be launched manually through
the EC2 console to run the app. At time of writing, the 'micro' instance size
was fine for our needs since the server had virtually no traffic.

### Credentials

You need to have a few credentials in place to be able to run the build script.
Add a `secrets/` directory to this project that contains the following files:

- google-sheets.json
- key-decoder.rb
- key-listing.txt
- tracker-basic-auth.json

Ask Edwin or Chris for these if you don't have them.

### Security Groups

When you launch the image you need to select the 'open-ssh-http-https' security
group in the AWS console, otherwise the machine's ports won't be open to accept
SSH connections or HTTP/S traffic.

### SSH Access

If you'd like to log onto the server via SSH, you need to build an image that
contains your public key. This can be added to `scripts/copy-public-key`. After
the image is built you should be able to log onto the machine with either of:

```sh
$ ssh field-backend.truefootprint.com
$ ssh tracker-backend.truefootprint.com
```

There is no need to set up key pairs when you launch the image through Amazon's
user interface as it already contains your SSH keys. You can 'Proceed without
a key pair'.

### Deployment

If you want to re-deploy either of the apps, you can either do that by
rebuilding the image from scratch which takes quite a while. Alternatively, you
can SSH onto the machine and run the following:

```sh
$ cd field-backend
$ sudo service field-backend stop
$ git pull --rebase
$ RAILS_ENV=production bundle exec rake db:migrate
$ sudo service field-backend start
```

You can use the same technique if you need to run rake tasks or access the rails
console.

### Domains

During the Packer build process, `scripts/setup-certbot` will ask you to set TXT
records against the truefootprint.com domain. This is so that letsencrypt can
issue SSL/TLS certificates for the applications. Otherwise, traffic would go
over HTTP and be unencrypted which is not appropriate for transmitting personal
data.

When this happens, you will be see something like this:

```
Please set the DNS TXT record for field-backend.truefootprint.com:
Set it to ABCDE12345 and wait for propagation..........
```

Go to AWS's Route53 service and find the subdomain and do as it says. A minute
or so later the build will continue once the change has been detected. This will
invalidate the previous certificate, so make sure you go through with the build
and launch the instance with the new certificate.

This definitely has room for improvement. It might be better to re-use existing
certificates rather issuing new ones.

### Elastic IP

Once the image is launched, the last thing you need to do to make it live is
re-assign the Elastic IP to the newly launched instance. You can do this through
the 'Elastic IPs' tab on the left on the EC2 service pages. The address of this
Elastic IP is 3.10.28.67. There is a DNS record that ties this to each of the
'field-backend' and 'tracker-backend' subdomains.

### Future plans

It might be better to move to a hosting platform like Heroku to simplify this
process, although it can get very expensive as our servers need to scale up.
For our basic needs, this was fine during development but to roll out to
production we'll need database backups, perhaps a staging server, perhaps a
continuous integration environment, etc. Feel free to throw everything here away
and do something else instead.

One thing to be a little bit careful of are the image uploads which required
some special setup in Nginx. See `scripts/setup-nginx` for more details.
