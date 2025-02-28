# InfernoMOO Leak of 2015 (Dockerized)

Welcome to the **InfernoMOO** "leak" of 2015! This project preserves an old LambdaMOO spinoff game using **Docker**.
I did this because nobody else has for some reason by now and a part of "game history" isn't being well preserved, so here ya go.

## Getting Started (Windows)

Since my host OS is **Windows**, these instructions focus on running the project on Windows. However, similar steps should work on other operating systems with minor modifications.

### Prerequisites

Ensure you have the following installed:

- [Docker for Windows](https://www.docker.com/products/docker-desktop) with Linux subsystem support (requires **Hypervisor**)
- [Git for Windows](https://gitforwindows.org/) (optional, but useful)
- [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/) for Telnet access

### Installation Steps

1. **Install Docker for Windows** with Linux subsystem support.
2. **Download this project** and extract it to a directory of your choice.
3. **Open a Windows Git Bash or Command Prompt** and navigate to the `inferno-compose` directory:
   ```sh
   cd path/to/inferno-compose
   ```
4. **Start the server using Docker Compose**:
   ```sh
   docker-compose up -d
   ```
5. **Watch the build and launch process** until it completes successfully.
6. **Connect to the server using PuTTY**:
   - Address: `localhost`
   - Port: `7779`
   - Protocol: **Telnet**
7. **Login as an admin (Wizperms)** using the 2015 setup:
   ```
   co Merlin imshana
   ```
8. If you encounter issues, check your **Docker logs** for errors and verify that the server is running:
   ```sh
   docker ps
   docker logs <container_id>
   ```

## Important Notes

### ⚠️ Considerations Before Hosting Publicly

1. **Hosting is functional but not fully secure.**
2. **Database backups are not managed automatically.**
   - You should learn how to **mount an external Docker volume** to store your database persistently. Also, you will need to modify the restart.sh to intelligently increment the save db file name and the load db file name every time the server reboots. Also, it seems like the game cannot handle you binding a database file to remote fileshare, so you will likely have to `cp` from the docker image to your database preservation drive periodically, which is unfortunate. 
3. **Email registration for new players is incomplete.**
   - In 2015, I attempted a rewrite to modernize it, but gave up. You will have to research how to hook the email registration functionality into the docker image to host this game on your own.
   - Update: I recently went down a rabbit hole configuring this and can give a few tips, see the appended section below.
4. **Security Enhancements Needed.**
   - The base Docker image likely needs **additional security measures**.
5. **Potential Exploits Exist.**
   - Some past security fixes were lost due to poor preservation efforts.
   - There may be known exploits (e.g., sending large 100MB strings to crash the server).

## Enjoy!

This is a preserved piece of MOO history. If you find ways to improve the image or address security issues, consider sharing your improvements with the community!

## Modding the game

1. purge.db is a little bit easier to work with, since it doesn't come with all the clutter.
2. Keep in mind that the server only allows a "thread" a certain number of ticks before it will time out. In order to avoid that, you should judiciously use $cu:sin() in "safe" places in verbs you write so that your code can "rest" and give back CPU time to other tasks the server wants to run. For instance, writing lines back to the client (such as to "draw" a GUI) is rather slow, so you should consider a $cu:sin() when doing a lot of operations like that (such as in a for-loop).
3. You can explore the game's code using `;#<id>`, `@show #<id>`, `@list #<id>:verbName`, `@properties #<id>`, `;#<id>.propertyName`, `@parents #<id>`, `@children #<id>`, and iteratively probe large things with something like `;for x in (properties(#<id>)) me:tell(x); $cu:sin(); endfor`

Unfortunately, "building utilities" exist, but aren't terribly friendly. There is built-in XML parsing if you dig, but it's not terribly easy to pass an XML string and make it create items or rooms, you're stuck building a helper function for that from what I can tell.

## Setting up email registration

Warning: This requires some technical know-how (or just GPT it I guess) to get working.
You will need to modify the Dockerfile to add the following things:

`ARG EMAIL`  
`ARG EMAIL_PASSWORD`  
`ARG MAIL_NAME`  
`ARG SMTP_DOMAIN`  
`ARG SMTP_PORT`  
...  
`RUN apt-get update && apt-get install -y postfix`  
...  
`COPY main.cf /etc/postfix/main.cf`  
`RUN dos2unix /etc/postfix/main.cf`  
...  
`RUN sh -c 'echo "root: ${EMAIL}" >> /etc/aliases' && \`  
    `sh -c 'echo "${MAIL_NAME}" >> /etc/mailname' && \`  
    `sh -c 'echo "[${SMTP_DOMAIN}]:${SMTP_PORT} ${MAIL_NAME}:${EMAIL_PASSWORD}" >> /etc/postfix/sasl_passwd' && \`  
    `postmap /etc/postfix/sasl_passwd && \`  
    `chmod 0600 /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db`  
...  
`RUN postconf -e "relayhost = [email-smtp.us-east-1.amazonaws.com]:587" \`  
`"smtp_sasl_auth_enable = yes" \`  
`"smtp_sasl_security_options = noanonymous" \`  
`"smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd" \`  
`"smtp_use_tls = yes" \`  
`"smtp_tls_security_level = secure" \`  
`"smtp_tls_note_starttls_offer = yes"`  
...  
`RUN apt update && apt install --reinstall ca-certificates`  

Create a .env file in the same directory as your docker-compose and populate the ARG variables with valid things:  
`EMAIL=<your to>@<your moo domain>.<com|net|etc>`  
`EMAIL_PASSWORD=mail pw`  
`PORT=25`  
`MAIL_NAME=mail username`  
`SMTP_DOMAIN=mail provider URL`  
`SMTP_PORT=587 or something`  

In the root directory of this project, you will need a main.cf file, which is a standard postfix .cf file, it shouldn't need any fancy configuration  
Look online for resources or http://www.postfix.org/COMPATIBILITY_README.html.  

Then, once you have done all that work, you can run the following on the MOO server:  
Considering the following "script" to run in your MOO, this configures the sending of registration emails to users:  
`;#72.site = "<your moo site domain>"`  
`;#72.postmaster = "<your to>@<your moo domain>.<com|net|etc>"`  
`;#72.port = <the port to your moo>`  
`;#72.MOO_name = <the name of your moo>`  
`;#72.usual_postmaster = <same as postmaster>`  
`;#72.password_postmaster = ""`  
`;#72.envelope_from = <same as postmaster>`  
`;"Alter the following ONLY if you use a non-standard website domain to host your moo"`  
`;#72.valid_host_regexp = "^%([-_a-z0-9]+%.%)+%(gov%|xyz%|edu%|com%|org%|int%|mil%|net%|%nato%|arpa%|[a-z][a-z]%)$" `  
`@program #72:return_address_for   this none this`  
`  ":return_address_for(player) => string of 'return address'. Currently inbound mail doesn't work, so this is a bogus address.";`  
`  who = args[1];`  
`  if (valid(who) && is_player(who))`  
`    if (who.wizard)`  
`      return "<!hard wire your own EMAIL HERE FOR TESTING!> (Game Admin)";`  
`    endif`  
`    return tostr(toint(who), "@", this.site, " (", who.name, ")");`  
`  else`  
`    return tostr($login.registration_address, " (non-player ", who, ")");`  
`  endif`  
`.`  

`;"Now do this to test!"`  
`;#72:sendmail("<email you want to test against>", "test", "test", "test")`  

You should successfully get an email. I was able to get mail registration working with AWS's simple mail provider (though it took a few hours of work, make sure you're using the correct region URL for the account you set up). But hopefully that helps people get started.
If the email does not arrive, you will need to install logging utils into the docker image and inspect `/var/logs/` for postfix, but you normally don't get helpful errors anyway, so just guess until it works!