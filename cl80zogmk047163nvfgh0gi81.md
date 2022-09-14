## Install Calibre Server on Ubuntu Server on 20.04

I've recently installed [Calibre](https://calibre-ebook.com/) on an Ubuntu server. This was not as easy as I was hoping! My use case for this is primarily to convert my existing ePub books to a Kindle supported Mobi format. Calibre has best in class ebook management tools so it's the perfect tool for the job.

In doing research for this I came across many articles but there was always a bit of murkiness as to what packages are required, and I hate getting a fresh clean version of Ubuntu unnecessarily cluttered. Then I came across an amazing Docker project - [linuxserver/calibre](https://hub.docker.com/r/linuxserver/calibre). _Spoiler alert_ - this is how you get Calibre on Ubuntu in a reliable and efficient manor.

### Step one - Set up your server

To get started, I'm using a [DigitalOcean droplet](https://m.do.co/c/0d1b8b43610c) (affiliate link) to run this but you can use any basic Ubuntu server.

For DigitalOcean you simply need to create a droplet using their UI. I'm using the cheapest one which is \$5/month. Make sure you select 20.04 as the operating system so you can get the new long term support version of Ubuntu.

From there, [setup your server](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-20-04) to get your server setup. Then simply [install Docker](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04) and your server is ready for the Calibre Docker container!

### Step two - Install the Docker Container

This part is straightforward once you've got all the steps done from above. Simply run the following command. **Note** - you may need to change the `PUID` and `GUID` variables. You can get those by running the command in your terminal `id ${USER}`.

```bash
docker create \
  --name=calibre \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=America/Los_Angeles \
  -p 8080:8080 \
  -p 8081:8081 \
  -v /path/to/data:/config \
  --restart unless-stopped \
  linuxserver/calibre

```

Be sure to change the part that says `/path/to/data` to whatever folder you'd like to have your Calibre data in. To start with I'd suggest using your home folder, similar to `/home/USER/calibre`. If that folder doesn't exist it will creat it for you, so definitely change it before you run the command.

### Step three - Using the Docker Container

Now you've got your Calibre server ready to use! If you want to use the utilities, you can use the `docker exec` utliity. For example, if you'd like to look a book up by ISBN, you can use the `fetch-ebook-metadata` command like this:

```bash
docker exec calibre fetch-ebook-metadata --isbn 0060850523
```

You may want to additionally expose this via a server such as NGinx or similar if you'd like to go to Calibre remotely. I'd suggest taking the time and diving into the [linuxserver/letsenccrypt](https://docs.linuxserver.io/images/docker-letsencrypt) Docker container for that.

Enjoy your Calibre server!