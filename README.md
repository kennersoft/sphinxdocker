sphinxdocker
============
Sphinx-beta in an Debian 10 Docker container! This container will require some configuration (which is really no big deal), but if you just want to pass in database connection parameters when starting the container, go try [this one](https://github.com/stefobark/QuickSphinx). It's probably the easiest way to get started playing around with Sphinx.

#####Build it:#####

```
docker build -t sphinx . 
```

#####Pull it:######

```
docker pull kennersoft/sphinxsearch:2.2.11
```

####Some Introduction####
```Dockerfile```  starts by adding the Debian 10 base image and installs Sphinx, creates some directories, ADDs our .sh files, and exposes port 9306. It'll take a bit to run through the steps but after some time, it should confirm a successful build. 

####Run the Container####

Run Sphinx in a 'detached' container (daemonized) like so:
```
docker run -p 9311:9306 -v /path/to/local/sphinx/conf:/etc/sphinxsearch/ -d sphinx ./indexandsearch.sh
```

The ```-p 9311:9306``` is opening port 9306 to port 9311 on the host machine. Open whatever port you've told searchd to listen to. Then, with -v we're sharing the ```/path/to/local/sphinx directory``` (which is the directory you're using for these docker files) with the container's ```/etc/sphinxsearch```. This is handy because we can now edit the Sphinx configuration file from the host machine. So, now, after running indexandsearch.sh, Sphinx should have a configuration file to work with.

####Persistent Index Data####
If you want index data to persist through container shutdowns, just add another ```-v /some/directory/:/var/lib/sphinx/data/``` to share a directory on your host machine with the Sphinx data directory within the container.

####.sh####
```indexandsearch.sh``` runs indexer using ```sphinxy.conf``` and then runs ```searchd.sh``` which... starts up searchd.
You might write your own configuration file and point Sphinx to your data source, or edit ```sphinxy.conf``` to match your setup. 

Also, it may be helpful to mention that in ```indexandsearch.sh```, I'm telling sphinx to index from ```/etc/sphinxsearch/sphinxy.conf``` instead of the default ```sphinx.conf``` (which you should take a look at if you want to see an expanded Sphinx configuration with a bunch of helpful comments about the various options.. or just go read the Sphinx docs). And, with ```lordsearch.sh```, we're starting searchd with ```bsphinx.conf``` (which is generated by running ```makelord.sh```).

####MySQL####
I'm using another container that's running MySQL. To connect to that MySQL container, I just ran ```sudo docker.io inspect <container id>``` got the Gateway address, and put that info into sphinxy.conf. You may also link containers. There's a bunch more fun stuff you can do with environment variables and such.

####Check it Out####
Now, let's make sure it's running:

```docker ps```

Perfect.

####Command Line Client####

Then, check the Sphinx instance inside the container with:

```mysql -h0 -P9311```

Try a SELECT statement or something.

####Realtime Indexing####
If we defined a realtime index in our configuration file, we could just run ```searchd.sh``` instead of ```indexandsearch.sh``` to just get searchd up and running. Although, ```indexandsearch.sh``` will work just as well.. It also starts searchd. 

####Playing with Distributed Search####
Sometimes it's good to shard your index, or you might want to do agent mirroring for HA/failover. For me, using docker to learn how this works was pretty nice. Convenient. I didn't have to worry about creating a unique PID, or a unique path for index/log files, which would be necessary if you were running multiple Sphinx instances on one machine. 

####An outline of what I did####

I started a bunch of Sphinx containers off of one image... many containers with unique names. To edit where searchd listens, and what will be indexed, for each container, I just edited ```sphinxy.conf``` before starting it:
```
docker run -p 9306:9306 -v /path/to/local/sphinx/conf:/etc/sphinxsearch/ --name sphinx1 -d stefobark/sphinx ./indexandsearch.sh
docker run -p 9307:9307 -v /path/to/local/sphinx/conf:/etc/sphinxsearch/ --name sphinx2 -d stefobark/sphinx ./indexandsearch.sh
docker run -p 9406:9406 -v /path/to/local/sphinx/conf:/etc/sphinxsearch/ --name sphinx3 -d stefobark/sphinx ./indexandsearch.sh
docker run -p 9407:9407 -v /path/to/local/sphinx/conf:/etc/sphinxsearch/ --name sphinx4 -d stefobark/sphinx ./indexandsearch.sh
```

This created some 'shards' and 'mirrors'.. Containers that have ports starting with ``93`` are all mirrors of each other, they contain the first 100 docs from our datasource. Those listening on ports starting with ```94``` are also mirrors of each other, they hold the next 100 docs.

From here, I started up 'lordsphinx':
```
docker run -p 9999:9999 -v /path/to/local/sphinx/conf:/etc/sphinxsearch/ --name lordsphinx -d stefobark/sphinx ./lordsearchd.sh
```

It held the 'distributed' index type, which maps to the other instances of Sphinx. 

####```makelord.sh``` makes a distributed index configuration file#####

Now, I'd like to figure out how to make this process even easier. This is how ```makelord.sh``` was born.

The motivation behind makelord.sh is to detect existing Sphinx containers, grab the port's they're listening on, and create a configuration file that the master node can use. So, running makelord.sh on the host machine will create a Sphinx configuration file for the master node called ```bsphinx.conf```. 

After this file is generated, start the last container, lordsphinx, with ```lordsearchd.sh``` (which will run Sphinx with ```bsphinx.conf```). Just an experiment. More messing around to do here.


###Exalmpe for using in docker-compose 
```
  sphinx_server:
    image: kennersoft/sphinxsearch:2.2.11
    container_name: spooxid6_sphinx
    volumes:
      - ./data/sphinxsearch/sphinxy.conf:/etc/sphinxsearch/sphinxy.conf
      - ./data/sphinxsearch:/var/lib/sphinx
    ports:
      - 9395:9395
    command: bash -c "/usr/bin/searchd -c /etc/sphinxsearch/sphinxy.conf && /usr/bin/indexer -c /etc/sphinxsearch/sphinxy.conf --rotate --all && tail -F /var/lib/sphinx/log/*.log"
```