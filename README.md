# valetudo-crontab
I am running a dustcloud build on my roborock s6 vacuum bot and on valetudo 0.6.1 the timers do not work.
So this is a little hacky workaround to get cronjobs for scheduling roborock s6 cleaning sessions.

First I would recommend to take a look at the WebGUI of your valetudo installation in the browser, and take a peek into the
developer console. Then navigate to the sources tab and locate the /services/api.services.js file to check out the available endpoints.
With these api endpoints we can easily set up a schedule on the robot itself.

We will call these API endpoints with curl, a cli tool to send GET/PUT/POST requests that is already installed on the robot.

## curl API call examples
Starting a full cleaning session:
```
curl --request PUT --url http://127.0.0.1/api/start_cleaning
```

Stopping any jobs the vacuum bot is doing:
```
curl --request PUT --url http://127.0.0.1/api/stop_cleaning
```

Sending the vacuum bot back home:
```
curl --request PUT --url http://127.0.0.1/api/drive_home
```

Clean zone id -1 and -4:
```
curl -H "Content-Type: application/json" -d '[-1, -4]' --request PUT --url http://127.0.0.1/api/start_cleaning_zones_by_id
```

To find the ids of your map zones open http://myrobotsip/api/zones in your browser and your will get a response from the API with 
a list of the zones you have configured on your map and their meta-data.

## scheduling these API calls with cron
Since every linux comes with a native scheduler called cron, we can leverage this to run our own timers without the webgui of valetudo.
First we need to connect to the robot via ssh and create the directory '/etc/crontabs'.
Normally we should run 'crontab -e' to edit our users crontab with vim, but since lots of people have issues using it, just create a new file 
called 'root' in the /etc/crontabs dir, our root user will be running the jobs.

## better use a cron generator if you don't know the syntax
Or you might end up telling your vacuum robot to start cleaning at 4:00am when you are trying to sleep, or even worse, 
tell it to start a cleaning job every 5 seconds because of a typo. I used this website just to be safe: https://crontab-generator.org/

## crontab examples
Here are some examples from my crontab that you can paste, be sure to understand how the syntax works though 
and test your commands before you make a new one to add.

```
# /etc/crontabs/root
# monday and thursday at 9am clean zones -3 and -2
0 9 * * 1 curl -H "Content-Type: application/json" -d '[-3,-2]' --request PUT --url http://127.0.0.1/api/start_cleaning_zones_by_id >/dev/null 2>&1
0 9 * * 4 curl -H "Content-Type: application/json" -d '[-3,-2]' --request PUT --url http://127.0.0.1/api/start_cleaning_zones_by_id >/dev/null 2>&1
# start at full clean sundays at noon
0 12 * * 0 curl --request PUT --url http://127.0.0.1/api/start_cleaning >/dev/null 2>&1
# clean zone -8 (pet area) sundays at 18:00
0 18 * * 0 curl -H "Content-Type: application/json" -d '[-8]' --request PUT --url http://127.0.0.1/api/start_cleaning_zones_by_id >/dev/null 2>&1
```

## start crond so that the scheduler ist actually running
Now that you have your crontab prepared, it's time to start the timers :)
Run `crond` manually if you just to want to test something. In order to make it permanent and start the scheduler on boot, edit your `/etc/rc.local`
and add `crond` right before the `exit 0` line and reboot.