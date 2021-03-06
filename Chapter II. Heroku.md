# Chapter II. Heroku

### Everything starts from account

Signing up to Heroku is a pretty straightforward process, everything is as usually as with any other service. 

You will probably have to verify your account. For example, if you want to host more than 5 apps, or use a custom domain. It's simple too and just requires you providing credit card information. To do that, go to your account settings:
![Heroku account settings](https://github.com/saasforge/book-on-deployment/blob/master/Illustrations/Heroku_account.png)

Then click the **Billing** tab and click the **Add credit card** button.

### Concept of dynos

Heroku introduced "dynos" which are described as "lightweight Linux containers". Full information you can find in [Heroku docs](https://devcenter.heroku.com/articles/dynos). We, especially those who are from the Windows world, can consider a dyno as a type of application (maybe it's wrong, but I can't find closer analogy). Dyno can be [free, hobby, standard, or performance](https://devcenter.heroku.com/articles/dyno-types). You always start from the free dyno. This type of dyno doesn't allow to link a custom domain and has other restrictions.

### Create an application and deploy

Let's create an application. We will use the web interface. Go to your dashboard and click the **New** button, then select **Create new app**:
![Heroku create new app](https://github.com/saasforge/book-on-deployment/blob/master/Illustrations/Heroku_create_app.png)

Enter the app's name and select a region.

We found the integration with Github very convenient and highly recommend to set it up. So, if you have your code in the Github, even in a private repository, you can connect it to the Heroku app and the code will be deployed automatically or when you are ready. To set up the integration, just open your app's page, click the **Deploy** tab, go to the **Deployment method** section, click **Connect to GitHub** and follow the instructions.

If you don't want to integrate your code with any remote repository, you will deploy manually using from your terminal. First, you need to download and install [Heroku CLI](https://devcenter.heroku.com/articles/heroku-command-line).

Then enter in your terminal:
```
heroku login
```
Enter your credentials. The next step is to create a remote repository because it's the only way to deploy your app. It means that you need to initialize your local folder. Do it as usually:

```
cd yourfolder
git init
heroku git:remote -a thenameofyourapp
```

Then commit your changes as you usually do:
```
git add .
git commit -am "First commit"
git push heroku master
```

When pushing to the Heroku repo, it may ask for credentials. And it's not the same as you already entered. What you should enter:
- username - leave blank
- password - Heroku auth token (that can be obtained via *heroku auth:token* command)

You can find all the details on it in the (Heroku docs)[https://devcenter.heroku.com/articles/git].

So, the pushing to the Heroku git actually runs deployment. But, before you deploy your code you need to do something more.

### Preparing app for deployment (Flask-based apps)

#### Gunicorn
Before pushing an application to the Heroku, create an empty file in the app's root folder and call it Procfile. Open it and add the following string into:

```
web: gunicorn module_name:variable_name
```

Find the full description of using gunicorn utility [in the official docs](http://docs.gunicorn.org/en/stable/run.html).

So, for example, you have app.py in the root folder which contains the application variable created with using factory approach:

```
application = create_app()
```

In this situation your command should look:

```
web: gunicorn app:application
```

Also, you need to install this package into your project. You can just add the corresponding line into your *requirements.txt* file, or  run these 2 commands:

```
pip install gunicorn
pip freeze > requirements.txt
```

#### Multiple buidpacks

If your app is written in Python but it uses webpack to install packages you need to run Node.js *before* running your Python app to prepare all NPM packages compiled and bundled. Heroku is smart enough to figure out that your app is in Python but it will not be able to guess to run *npm* command before. So you need to do 2 things:

1. Tell webpack to run the corresponding configuration (we assume you already have separate webpack config files for development and production). To do so, add the following strings into your *package.json* file:

```
"scripts": {
    "dev": "webpack --config webpack.dev.js",
    "prod": "webpack --config webpack.prod.js",
    "heroku-prebuild": "npm install",
    "heroku-postbuild": "webpack --config webpack.$env.js"
},
```

So, basically, it means that you install all packages and then ask webpack to run the corresponding configuration.

:warning: **Don't forget to provide a proper environment variable *env*, in our case it's *dev* or *prod*.**

2. Tell Heroku that you want to run Node.js first, then Python. It can be done using [buildpacks](https://elements.heroku.com/buildpacks). To do so, open your app's page, click the **Settings** tab, scroll a little bit and find the **Buildpacks** section. Add buildpack, Node.js, and Python:

![Heroku buildpacks](https://github.com/saasforge/book-on-deployment/blob/master/Illustrations/Heroku_buildpack.png)

### Custom domain and SSL

Start from registering your domain at some domain provider. We use Namecheap because they have a really great service (and great prices). 

Then you need to upgrade your dyno. In your application's page click the **Resources** tab, then select at least "Hobby". You will be asked to enter your credit card information if you didn't do so.

After you're done, open your application's page, click the **Settings** tab, scroll down to the **Domains and certificates** section. Click the **Add domain** button. Enter your registered domain, then open your domain provider's console and create the corresponding CNAME. Please refer the [Heroku docs](https://devcenter.heroku.com/articles/custom-domains#configuring-dns-for-subdomains) to see what you should provide in CNAME, but basically, it's your app Heroku URL like *yourfullappurl.herokudns.com.*

:warning: **The URL you should enter in your domain provider is not your heroku app URL like *yourapp.herokuapp.com*.**

Below is the example of providing data in Namecheap console:

![Namecheap console custom domain](https://github.com/saasforge/book-on-deployment/blob/master/Illustrations/Heroku_custom_domain.png)

After you're done, come back to the Heroku console and click the **Refresh status** button (it may take some time to update DNS records).

:point_right: **SSL certificate is created and updated automatically by Heroku.**

## Programmatic deployment (Python)

Heroku doesn't have a Python interface, so we just use their API. The code below is in Python:

```
heroku_session = None
heroku_url = 'https://api.heroku.com'

def get_heroku_credentials():
    global heroku_session
    if heroku_session is None:
        heroku_headers = {'Accept': 'application/vnd.heroku+json; version=3.cedar-acm',
            'Content-Type': 'application/json'}

        heroku_session = requests.session()
        heroku_session.headers.update(heroku_headers)
        heroku_session.auth = ('', 'API_key')
        result_auth = heroku_session.get(heroku_url + '/account/rate-limits')
        
def get_response_json(response):
    res_json = response.content.decode('utf8').replace("'", '"')
    return json.loads(res_json)
    
def deploy_to_heroku(env_name, app_id, app_name):
    get_heroku_credentials()

    # Create app build
    tar_url = 'https://s3.your_region.amazonaws.com/apps_bucket/{0}.tar.gz'.format(env_name)
    url = heroku_url + '/apps/' + app_id + '/builds'
    payload = {
        'source_blob': {
            'url': tar_url
        },
        'buildpacks': [
            {
                'name': 'heroku/nodejs'
            },
            {
                'name': 'heroku/python'
            }]
    }

    res_deploy = heroku_session.request('post', url, data=json.dumps(payload))
    if res_deploy.ok:
        res_data = get_response_json(res_deploy)
    return {
        'result': res_deploy.ok,
        'status': res_data['status'],
        'app_url': 'https://{0}.herokuapp.com/'.format(app_name),
        'last_build_id': res_data['id']
    }
```

The first step is, as usual, getting credentials. You have your API key (go to your account page, then click the **Account** tab and scroll till the **API key** section). Using your API key, you create a session and then use it to do other operations.

Heroku can create a working application only from the archived app, and this archive should be tar. So, assuming you somehow managed to create tar.gz archive from your app and put it in some public access (we use Amazon S3 for this purpose).

When you send the POST request to *apps/app_name/builds* Heroku endpoint you pass the URL to your tar.gz file. If deploy is done successfully, the response has the *ok* property what should be True, else False. The response.content is a text and it has to be parsed to JSON if you want to get all its values.
