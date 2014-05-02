Quickstart
==========

This addon extends [knplabs/github-api](https://packagist.org/packages/knplabs/github-api) and adds support for oauth connection to Github,
so you can seamlessly integrate your application with and provide login through Github.



Installation
-----------

The best way to install Kdyby/Github is using  [Composer](http://getcomposer.org/):

```sh
$ composer require kdyby/github:@dev
```

With Nette 2.1 and newer, you can enable the extension using your neon config.

```yml
extensions:
	github: Kdyby\Github\DI\GithubExtension
```

If you're using older Nette, you have to register it in `app/bootstrap.php`

```php
Kdyby\Github\DI\GithubExtension::register($configurator);

return $configurator->createContainer();
```



Minimal configuration
---------------------

This extension creates new configuration section `github`, the absolute minimal configuration is app ID and secret.
You also might wanna provide default required permissions.

```yml
github:
	appId: "1234567890"
	appSecret: "e807f1fcf82d132f9bb018ca6738a19f"
	permissions: [user:email]
```

And that's all.



Calling Github API
--------------------

Because this addon extends the original SDK, you can have a look at what `api()` does and more on their documentation,
which as [in the github repository KnpLabs/php-github-api](https://github.com/KnpLabs/php-github-api/blob/master/doc/index.md)



Authentication
--------------

Hey, why there is no authenticator? Well, it's because the Github login sustains of several HTTP redirects,
and as you might have noticed, those don't fit well into PHP objects.

Then how are we going to login to Github you may ask? Easily! There is a component for that!

```php
use Kdyby\Github\UI\LoginDialog;

class LoginPresenter extends BasePresenter
{

	/** @var \Kdyby\Github\Client */
	private $github;

	/** @var UsersModel */
	private $usersModel;

	/**
	 * You can use whatever way to inject the instance from DI Container,
	 * but let's just use constructor injection for simplicity.
	 *
	 * Class UsersModel is here only to show you how the process should work,
	 * you have to implement it yourself.
	 */
	public function __construct(\Kdyby\Github\Client $github, UsersModel $usersModel)
	{
		parent::__construct();
		$this->github = $github;
		$this->usersModel = $usersModel;
	}


	/** @return LoginDialog */
	protected function createComponentGithubLogin()
	{
		$dialog = new LoginDialog($this->github);

		$dialog->onResponse[] = function (LoginDialog $dialog) {
			$github = $dialog->getClient();

			if ( ! $github->getUser()) {
				$this->flashMessage("Sorry bro, github authentication failed.");
				return;
			}

			/**
			 * If we get here, it means that the user was recognized
			 * and we can call the Github API
			 */

			try {
				$me = $github->api('me')->show();

				if (!$existing = $this->usersModel->findByGithubId($github->getUser())) {
					/**
					 * Variable $me contains all the public information about the user
					 * including github id, name and email, if he allowed you to see it.
					 */
					$existing = $this->usersModel->registerFromGithub($me);
				}

				/**
				 * You should save the access token to database for later usage.
				 *
				 * You will need it when you'll want to call Github API,
				 * when the user is not logged in to your website,
				 * with the access token in his session.
				 */
				$this->usersModel->updateGithubAccessToken($github->getUser(), $github->getAccessToken());

				/**
				 * Nette\Security\User accepts not only textual credentials,
				 * but even an identity instance!
				 */
				$this->user->login(new \Nette\Security\Identity($existing->id, $existing->roles, $existing));

				/**
				 * You can celebrate now! The user is authenticated :)
				 */

			} catch (\Github\Exception\ExceptionInterface $e) {
				/**
				 * You might wanna know what happened, so let's log the exception.
				 *
				 * Rendering entire bluescreen is kind of slow task,
				 * so might wanna log only $e->getMessage(), it's up to you
				 */
				Debugger::log($e, 'github');
				$this->flashMessage("Sorry bro, github authentication failed hard.");
			}

			$this->redirect('this');
		};

		return $dialog;
	}

}
```

And in template, you might wanna render a link to open the login dialog.

```smarty
{* By the way, this is how you do a link to signal of subcomponent. *}
<a n:href="githubLogin-open!">Login using github</a>
```

Now when the user clicks on this link, he will get redirected to Github authentication page,
where he can allow your page or decline it. When he confirms the privileges your application requires,
he will be redirected back to your website. Because we've used Nette components,
he will be redirected to a signal on the `LoginDialog`, that will invoke the event
and your `onResponse` callback will be invoked. And from there, it's a child play.



Best practices
--------------

Please keep in mind that the user can revoke the access to his account literary anytime he want's to.

Therefore you must wrap every github api call with `try catch`

```php
try {
	// ...
} catch (\Github\Exception\ExceptionInterface $e) {
	// ...
}
```

and if it fails, try requesting the list of your permissions.
This will tell you if the user revoked your application entirely, or he only removed some of your privileges.

And if he revokes your application, drop the access token, it will never work again, you may only acquire a new one.