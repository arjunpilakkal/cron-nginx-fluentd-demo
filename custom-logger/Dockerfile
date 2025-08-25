FROM alpine:3.20

# busybox includes crond in Alpine; no extra cron package needed
# Create log directory and copy cron bits
RUN mkdir -p /var/logs
COPY crontab.txt /etc/crontabs/root
COPY cronjob.sh /cronjob.sh
RUN chmod +x /cronjob.sh

# Run cron in foreground with log level 2
CMD ["crond", "-f", "-l", "2"]
