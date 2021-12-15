FROM ubuntu:20.04

RUN apt-get update && apt-get install -y inotify-tools

CMD inotifywait -r -m /mnt
