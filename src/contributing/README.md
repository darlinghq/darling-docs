# Contributing

If you are familiar with how GitHub Pull Requests work, you should feel right at home contributing to Darling. 

## Fork the repository

Locate the repository that you made changes in on GitHub. The following command can help. 
	
```
cd darling/src/external/less
git remote get-url origin
https://github.com/darlinghq/darling-less.git
```
If it is an `https` scheme then you can paste the URL directly into your browser. 

Once at the page for the repository you made changes to, click the *Fork*
button. GitHub will then take you to the page for your fork of that repository.
The last step here is to copy the URL for your fork. Use the Clone or download
button to copy it.

# Commit and push your changes

Create and check out a branch describing your changes. In this example, we will
use `reinvent-wheel`. Next, add your fork as a remote.

```
git remote add my-fork git@github.com:user/darling-less.git
```

After this, push your commits to your fork. 

```
git push -u my-fork reinvent-wheel
```

The `-u my-fork` part is only necessary when a branch has never been pushed to your fork before. 

# Submit a pull request

On the GitHub page of your fork, select the branch you just pushed from the
branch dropdown menu (you may need to reload). Click the *New pull request*
button. Give it a useful title and descriptive comment. After this, you can
click create.

After this, your changes will be reviewed by a Darling Project member.
