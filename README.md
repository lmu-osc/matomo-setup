# OSC Matomo Analytics Set-Up

* Repo contains the docker compose we use for our Matomo analytics
* First commit in this repo is just the original version we used
* However, migrated to a more complete version based off of the instructions from Matomo at https://matomo.org/faq/how-to-install/install-matomo-with-docker/
* Slightly adjusted the instructions there to match to our naming conventions and existing volumes (i.e. where data is stored)
* on our server, needed to run `chmod -R 777 ./matomo/tmp` after the first attempt to run `docker compose up -d` resulted in the error "The directory "/var/www/html/tmp/cache/tracker/" does not exist and could not be created." Stopping the docker compose network, running that command, and then starting it up again resolved the problem.
    - this was actually a slightly larger problem, and i wound up giving `chmod -R 777 ./matomo/` as, for example, I ran into an issue that matomo/config was not writeable: "The Matomo configuration file (config/config.ini.php) is not writable, some of your changes might not be saved. Please change permissions of the config file to make it writable."
* Additional misc. note: on looking at the System Check, there was a notice that the last cron backup was ~40 days old (from a time I manually ran it on the server), but I was expecting the cron docker service to take care of it. From inspecting the docker logs, `docker logs matomo-cron`, I could see that the archiving just took ~6 minutes to complete after it was triggered. After that, system check came back all clear.
* Note: on the System > General Settings > Archiving Settings, we have "Archive reports when viewed from the browser" set to "No" (done manually in the browser) because the cron job now does this automatically on a schedule
* The .env-example file shows the additional environment fields (at minimum) that should be included.

* On our server, one should pay attention that they have the correct permissions of the folder, that their SSH connection with GitHub is set-up, etc. in order to be able to `git pull` changes to this repo