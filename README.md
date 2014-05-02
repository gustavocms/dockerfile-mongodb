dockerfile-mongodb
==================



# docker run -p 11211 memcached

sudo docker run \
  -P -name rs1_srv1 \
  -d gustavocms/mongodb \
  --replSet rs1 \
  --noprealloc --smallfiles
