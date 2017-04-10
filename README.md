## Heroku Review App Guide for Public and Private Dependencies
---

### What are Review Apps?
> Review apps run the code in any GitHub pull request in a complete, disposable app on Heroku. Each review app has a unique URL you can share. ([learn more](https://devcenter.heroku.com/articles/github-integration-review-apps))

---

#### Basic Setup (Public Dependencies)

###### Heroku Setup:
1. Create a new Heroku App
1. While viewing your app and under the "Deploy" tab, click "New Pipeline...".
1. Name your pipeline, and click "Create Pipeline".
1. Click "Connect to GitHub"
1. On the next screen, search the repo you want to enable your review apps on and click "Connect".
1. Click "Pipeline" in navigation.
1. Enable Review Apps through the following instructions:

Heroku requires a basic `app.json` file ([learn more](https://devcenter.heroku.com/articles/app-json-schema)) in your project's **default branch** to enable review apps within the Heroku Dashboard.

```json
{
  "name": "My Cool App",
  "description": "This app does cool things."
}
```

Heroku will attempt to auto-detect the type of project you have, however it is recommended that you declare the buildpack ([learn more](https://devcenter.heroku.com/articles/buildpacks)) ahead of time by adding the following to `app.json`.

**NOTE**: The example buildpack will automatically run `npm install` for you during setup.

```json
{
  "buildpacks": [
    {
      "url": "heroku/nodejs"
    }
  ]
}
```

**NPM CONFIG**: By default, Heroku will only install a project's regular dependencies. If your deployment process requires use of its _devDependencies_, then the following must be added to `app.json`.

```json
{
  "env": {
    "NPM_CONFIG_PRODUCTION": "false",
    "NODE_ENV": "development"
  }
}
```

In order to serve a static web site from Heroku, you can add the `http-server` [package](https://www.npmjs.com/package/http-server) to your project via the below command.

```sh
$ npm install http-server --save-dev
```

Then add a new script to `package.json` so the server can be launched.

```json
{
  "scripts": {
    "heroku-init": "http-server ./ -p ${PORT:=5000} -c-1"
  }
}
```

**NOTE**: This is a good place to add other deploy steps you may need as well. For example, if you need to run a gulp command before serving the site.

```json
{
  "scripts": {
    "heroku-init": "gulp custom-command && http-server ./ -p ${PORT:=5000} -c-1"
  }
}
```

The final code step is to create a `Procfile` ([learn more](https://devcenter.heroku.com/articles/procfile)) in the project root. This file will run your project's setup after installing the required dependencies.

```sh
web: npm run heroku-init
```

###### Finalize Heroku Setup
1. From your created pipeline, click "Enable Review Apps...".
1. Enable "Create new review apps for new pull requests automatically". If you don't, your team will need to log into Heroku to create them manually for each pull request.
    * This will affect your overall [cost](https://devcenter.heroku.com/articles/github-integration-review-apps#review-apps-management-and-costs).
1. You may want to enable "Destroy stale review apps automatically". This will shut down the Review App if there have been no commits to the branch for 5 days.
    * This will affect your overall [cost](https://devcenter.heroku.com/articles/github-integration-review-apps#review-apps-management-and-costs).
1. Click "Enable"

---

#### Advanced Setup (Private Dependencies using SSH)

If your project relies on private dependencies, you will need to jump through a few hoops regarding authentication. This section of the guide builds on the **Basic Setup** section.

The first step is to generate an SSH Key Pair so Heroku can communicate with your private repos.

```sh
$ ssh-keygen -t rsa -b 4096 -C "heroku/deploy/my_cool_app"
```

You will then be prompted for the location to save your keys. Take care not to overwrite any current keys when setting the path. **DO NOT** enter a passphrase in the next step, as there will be no user present when it is needed.

Next we'll add the public half of your newly generated key (`new_key.pub`) to GitHub. This can be accomplished by adding it to a specific repo as a deploy key or a specific user. This guide only supports the user approach, and it is recommended that you create a custom user so control can be shared among active developers.

* User
    * Pro: Can be used across multiple projects
    * Con: Key will have write access
    * How:
        * Navigate to [SSH and GPG Keys](https://github.com/settings/keys) on GitHub.
        * Click "New SSH Key"
        * Enter a descriptive _title_. e.g. "Heroku My_Cool_App Review Apps Deploy Key"
        * Copy the contents of `new_key.pub` into the _key_ field
        * Click "Add SSH Key"
* Deploy Key
    * Pro: More secure by only allowing read access
    * Con: Can only be used for one project
    * How:
        * Navigate to private repo for dependency
        * Click on "Settings"
        * Click "Deploy Keys"
        * Click "Add Deploy Key"
        * Enter a descriptive _title_, e.g. "Heroku My_Cool_App Review Apps Deploy Key"
        * Copy the contents of `new_key.pub` into the _key_ field
        * Click "Add Key"

Now that you've setup your public key, we'll add the private key to Heroku as an environment variable so it can be easily and securely managed. In your terminal, run the following to base64 encode and copy the private key.

MacOS
```sh
$ cat path/to/new_key | base64 | pbcopy
```

Now follow these steps to add it to Heroku.

* In Heroku, click on your app.
* Under production, click on the app title.
* Click on the "Settings" navigation menu.
* Click "Reveal Config Vars"
* Enter `GITHUB_SSH_KEY` for the _KEY_ field.
* Enter your recently copied base64 encoded key into the _VALUE_ field.
* Click "Add"

You can now delete your local copies of `new_key.pub` & `new_key`.

In order to use your new environment variable, you'll need to tell Heroku to share it with your app. Do this by adding it to the `app.json` file.

```json
{
  "env": {
    "GITHUB_SSH_KEY": {
      "required": true
    }
  }
}

```

Now that we have access to the private key, it's time to build two scripts to take advantage of it. The first script will be called by Heroku just before running NPM ([learn more](https://devcenter.heroku.com/articles/nodejs-support#heroku-specific-build-steps)), and will setup the SSH key for use. This script will be called `heroku-prebuild.sh`, and will be stored in `/bin`.

*The following snippet is attributed to [fiznool](http://stackoverflow.com/users/1171775/fiznool) via [Stack Overflow](http://stackoverflow.com/a/29677091).*

```sh
#!/bin/bash
# Generates an SSH config file for connections if a config var exists.
# Credit to fiznool via http://stackoverflow.com/a/29677091

if [ "$GITHUB_SSH_KEY" != "" ]; then
  echo "Detected SSH key for git. Adding SSH config"
  echo ""

  # Ensure we have an ssh folder
  if [ ! -d ~/.ssh ]; then
    echo "Creating directory"
    mkdir -p ~/.ssh
    chmod 700 ~/.ssh
  fi

  # Load the private key into a file.
  echo "Decoding key"
  echo $GITHUB_SSH_KEY | base64 --decode > ~/.ssh/deploy_key

  # Change the permissions on the file to
  # be read-only for this user.
  echo "Setting key as read-only"
  chmod 400 ~/.ssh/deploy_key

  # Setup the ssh config file.
  echo ""
  echo "Creating Config >>"
  echo -e "Host github.com\n"\
          "HostName github.com\n"\
          "User git\n"\
          "StrictHostKeyChecking no\n"\
          "IdentityFile ~/.ssh/deploy_key\n"\
          > ~/.ssh/config
fi

```

The second script will clean up the SSH files to keep the environment secure. This script will be called `heroku-postbuild.sh`, and will be stored in `/bin`.

*The following snippet is attributed to [fiznool](http://stackoverflow.com/users/1171775/fiznool) via [Stack Overflow](http://stackoverflow.com/a/29677091).*

```sh
#!/bin/bash
# Removes SSH key after use
# Credit to `fiznool` via http://stackoverflow.com/a/29677091

if [ "$GITHUB_SSH_KEY" != "" ]; then
  echo "Cleaning up SSH config"
  echo ""

  # Now that npm has finished running, we shouldn't need the ssh key/config anymore.
  # Remove the files that we created.
  rm -f ~/.ssh/config
  rm -f ~/.ssh/deploy_key

  # Clear that sensitive key data from the environment
  unset GITHUB_SSH_KEY
fi
```

Finally, to tie everything together you'll link these two scripts to your `package.json` with special hooks Heroku calls during the build process. The `bash` command is required to be called prior to the scripts so Heroku can provide a shell environment to run them in.

```json
{
  "scripts": {
    "heroku-prebuild": "bash bin/heroku-prebuild.sh",
    "heroku-postbuild": "bash bin/heroku-postbuild.sh"
  }
}
```

**NOTE**: If you are using another dependency manager such as  [Bower](https://bower.io) with private dependencies as well, you will need to run it during the _postinstall_ step ([learn more](https://docs.npmjs.com/misc/scripts)) in NPM in order to use the SSH key.

```json
{
  "scripts": {
    "postinstall": "bower install"
  }
}
```
