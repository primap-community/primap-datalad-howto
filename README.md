# PRIMAP datalad repositories

## Table of Contents

* [PRIMAP datalad repositories](#primap-datalad-repositories)
   * [Overview](#overview)
   * [One-time cloudflare setup](#one-time-cloudflare-setup)
      * [For internal developers](#for-internal-developers)
      * [For external collaborators](#for-external-collaborators)
   * [Create a new repository](#create-a-new-repository)
   * [Move an existing repository to R2](#move-an-existing-repository-to-r2)

## Overview

We're hosting primap datasets using github for metadata storage
and cloudflare R2. It essentially has three components:
* The github storage holds a git repository with the usual branches containing un-annexed 
  files like code files and other small text files and symlinks for data files. It also
  carries the git-annex branch with metadata (content hashes) for the data files, but
  crucially not the content of the actual data files. The github repo can be public or
  private, as configured on github.
* Cloudflare R2 contains all data files (using their hashed content as filename) in an
  object-orient storage. For each repository there is one `bucket` in R2. To access
  a bucket (both for reading and writing), you need to have an access key.
* Cloudflare R2 also offers the option to publish a `bucket` so that it is readable
  without an access key (this is not the default).

Taken together, this enables us to use github to manage the metadata (e.g. using branches
and pull requests for collaboration) while the data is a relatively cheap and efficient
object storage, all publicly available if desired.

## One-time cloudflare setup

### For internal developers

Internal developers get the rights to create new datasets and read from and write to
every public and private repository. For that, you'll need a cloudflare account for
web access, as well as an API key to access repositories.

First, get an account on cloudflare. Speak to Mika or Jared.

Once you have an account on cloudflare, you need to generate the API keys:

1. Go to https://dash.cloudflare.com/
3. Log in.
4. If it asks, choose the account "Climate Resource".
5. In the left menu, choose "R2 Object Storage".
6. On the right, select "Manage R2 API tokens".
7. Press "Create API token".
8. Use the name `{initials}-rw`, where `{initials}` are initials you identify with, like
   `mp` for Mika Pflüger.
9. As Permissions, choose "Object Read & Write".
10. For "Specify bucket(s)" select "Apply to all buckets in this account".
11. As TTL specify "Forever" for ease of use, or specify "1 year" for higher security
    (but you will need to regenerate the token every year).
12. Do not specify anything in "Client IP Filtering".
13. Presse "Create API Token"

Now, this will display (once only) a page with the API keys. In particular, you need
the "Access Key ID" and the "Secret Access Key ID". Save them somewhere safe.

To work with the API keys easily without having to type them out all the time, you
can export them using a function defined in your `.bashrc`. In your `~/.bashrc` file,
add this to the end:
```shell
primap_datalad_creds () {
    export AWS_ACCESS_KEY_ID="access-key-id"
    export AWS_SECRET_ACCESS_KEY="secret-access-key-id"
}
```
After reopening your terminal, you can now execute `primap_datalad_creds` once and
will be able to run any datalad commands which need access to the R2 until you close
the terminal. Alternatively, if you do not use any other AWS api keys and don't want
to be bothered to execute a function each time you open a new shell before you can
use datalad properly, you can also just add the two `export` lines to your `.bashrc`
without the surrounding function definition so that they get run everytime and the
environment variables are always available.

### For external collaborators

External collaborators get read or read-and-write rights to specific repositories,
but not the right to create new repositories. As such, they only need an API key, not a
cloudflare account.

To set an API key up for an external collaborator:

1. Go to https://dash.cloudflare.com/
3. Log in.
4. If it asks, choose the account "Climate Resource".
5. In the left menu, choose "R2 Object Storage".
6. On the right, select "Manage R2 API tokens".
7. Press "Create API token".
8. Use the name `external-{initials}-rw`, where `{initials}` are initials for the
   external collaborator.
9. As Permissions, choose "Object Read & Write" if you want to give write access,
   otherwise choose "Object Read only" (only useful for private repos, otherwise
   they should just use public access).
10. For "Specify bucket(s)" select "Apply to specific buckets only" and select buckets
    that the collaborator should have access to.
11. As TTL specify "Forever" for ease of use, or specify "1 year" for higher security
    (but you will need to regenerate the token every year).
12. Do not specify anything in "Client IP Filtering".
13. Presse "Create API Token"

Now, this will display (once only) a page with the API keys. In particular, you need
the "Access Key ID" and the "Secret Access Key ID". Send them to the collaborator.

To work with these API keys, the collaborator will likely want to set up their
`~/.bashrc` as explained above in the section for internal developers.

## Create a new repository

To create a new repository, first create the cloudflare R2 bucket and optionally enable
public access:
1. Go to https://dash.cloudflare.com/
2. Log in.
3. If it asks, choose the account "Climate Resource".
4. In the left menu, choose "R2 Object Storage".
5. Press the blue "Create bucket" button.
6. Decide on a dataset name, then use `primap-datalad-{name}` as the bucket name, where
   you replace `{name}` by the dataset name like `unfccc-di`, so that the resulting
   bucket name is something like `primap-datalad-unfccc-di`.
7. For "Location", choose "Automatic" and provide a "location hint" for
   "Western Europe (WEUR)".
8. For the "Default storage class", choose "Standard" (should be the default).
9. Hit "Create bucket" at the bottom.
10. The next steps are optional, and only necessary if you want public access to your
    repository.
11. The bucket overview loads, here navigate to the "Settings" tab, there you find
    under the "Public Access" heading in the "R2.dev subdomain" section a button
    "Allow access" - press it.
12. Type "allow" to confirm that you want public access, then press "Allow" again.
13. Copy the "Public R2.dev Bucket URL", you'll need it later.

Now, create your datalad repository. In the terminal, navigate to the location where
you want your local clone of the datalad repository (without creating the new directory
yet). Run:
```shell
datalad create -c text2git $name
cd $name
```
where you replace `$name` by the dataset name like `unfccc-di`.

Next, we'll add R2 as a sibling (i.e. publication target) to the new datalad repository.
We have to use git-annex directly to do this. If you add a bucket with public access,
use:
```shell
primap_datalad_creds  # skip if you set up your .bashrc to always inject the secrets
git annex initremote public-r2 type=S3 encryption=none signature=v4 region=auto protocol=https \
    autoenable=true \
    bucket=primap-datalad-$name \
    host=2aa5172b2bba093c516027d6fa13cdc8.r2.cloudflarestorage.com \
    publicurl=$publicurl
```
where you replace `$name` by the dataset name like `unfccc-di` and `$publicurl` by the
public URL you copied when creating the cloudflare R2 bucket. If you didn't copy it,
you can find it on the bucket's page in the settings under the heading "Public Access"
in the section "R2.dev subdomain" at the entry "Public R2.dev Bucket URL". Copy it fully,
it should look something like `https://pub-lotsofcharacters.r2.dev`.

Now, the output of `datalad siblings` should look like this:
```
.: here(+) [git]
.: public-r2(+) [git]
```

Now, we'll add github as a sibling to the repo. This can be done using datalad directly:
```shell
datalad create-sibling-github primap-community/$name \
    --publish-depends public-r2 \
    --access-protocol ssh
```
where you replace `$name` again by the dataset name like `unfccc-di`.

Now, the output of `datalad siblings` should look like this:
```
.: here(+) [git]
.: public-r2(+) [git]
.: github(-) [git@github.com:primap-community/github-r2-datalad-test.git (git)]
```

Finally, push the dataset to github, which will automatically push to R2 as well
because we configured it as a publication dependency.


## Move an existing repository to R2

To move an existing repository, first create the cloudflare R2 bucket and optionally enable
public access:
1. Go to https://dash.cloudflare.com/
2. Log in.
3. If it asks, choose the account "Climate Resource".
4. In the left menu, choose "R2 Object Storage".
5. Press the blue "Create bucket" button.
6. Use `primap-datalad-{name}` as the bucket name, where
   you replace `{name}` by the dataset name like `unfccc-di`, so that the resulting
   bucket name is something like `primap-datalad-unfccc-di`.
7. For "Location", choose "Automatic" and provide a "location hint" for
   "Western Europe (WEUR)".
8. For the "Default storage class", choose "Standard" (should be the default).
9. Hit "Create bucket" at the bottom.
10. The next steps are optional, and only necessary if you want to maintain public 
    access to your repository.
11. The bucket overview loads, here navigate to the "Settings" tab, there you find
    under the "Public Access" heading in the "R2.dev subdomain" section a button
    "Allow access" - press it.
12. Type "allow" to confirm that you want public access, then press "Allow" again.
13. Copy the "Public R2.dev Bucket URL", you'll need it later.

Now, we'll add R2 as a sibling (i.e. publication target) to the existing datalad repository.
We have to use git-annex directly to do this. If you add a bucket with public access,
use:
```shell
primap_datalad_creds  # skip if you set up your .bashrc to always inject the secrets
git annex initremote public-r2 type=S3 encryption=none signature=v4 region=auto protocol=https \
    autoenable=true \
    bucket=primap-datalad-$name \
    host=2aa5172b2bba093c516027d6fa13cdc8.r2.cloudflarestorage.com \
    publicurl=$publicurl
```
where you replace `$name` by the dataset name like `unfccc-di` and `$publicurl` by the
public URL you copied when creating the cloudflare R2 bucket. If you didn't copy it,
you can find it on the bucket's page in the settings under the heading "Public Access"
in the section "R2.dev subdomain" at the entry "Public R2.dev Bucket URL". Copy it fully,
it should look something like `https://pub-lotsofcharacters.r2.dev`.

Now, the output of `datalad siblings` should look like this:
```
.: here(+) [git]
.: public-r2(+) [git]
.: origin(-) [https://github.com/mikapfl/unfccc_di_data.git (git)]
.: datalad-archives(+) [datalad-archives]
.: ginhemio-storage(+) [https://gin.hemio.de/CR/unfcc_di_data (git)]
.: ginhemio(+) [https://gin.hemio.de/CR/unfcc_di_data (git)]
```
Note that your github sibling might not be named "origin" and you might not have
the "datalad-archives" sibling at all and you might only have one of "ginhemio" and
"ginhemio-storage". This all depends on the prior hosting history of the dataset and
might therefore differ between datasets. Important is only that you at this stage still
have one ginhemio sibling and the github sibling.

Now, we'll add a publication dependency on public-r2 to the github remote and remove
the publication dependency on ginhemio:
```shell
datalad siblings configure -s $github_sibling_name --publish-depends public-r2
```
Replace `$github_sibling_name` with the name of your github sibling (usually, `github`
or `origin`).

Now, push the dataset to github, which will automatically push to R2 as well
because we configured it as a publication dependency. This might take a while because
it transfers all data:
```shell
datalad push --to $github_sibling_name
```

Finally, remove the obsolete ginhemio siblings:
```shell
datalad siblings remove -s ginhemio
datalad siblings remove -s ginhemio-storage
```
if you only have one ginhemio sibling, only remove this one. Also, we have to tell
git-annex to never enable the storage sibling and not try to fetch data from ginhemio
any more:
```shell
git annex configremote ginhemio-storage autoenable=false
git annex dead ginhemio-storage
```
and push the results again:
```shell
datalad push --to $github_sibling_name
```

## Add an existing R2 sibling to your local repository

If someone already set up the R2 sibling, we still need 
to add it to our local repository. When we pull from GitHub our local repository
is already aware of the sibling, but we might not be able to see it yet.

First we check if the R2 sibling has already been added with:

```shell
git annex whereis path/to/some/file
```

Choose a file that ist stored in the datalad remote (e.g. .nc or .csv files). We should now see a list of remotes where the file is stored. One of them will have the name `public-r2`
and show the url of the R2 bucket. Note that it could also have another name, if the remote wasn't set up
as described above.

If the command does nothing we can try the same for all files and interrupt the process after a few files:

```shell
git annex whereis
```

If we want to see the name of the bucket we can use:

```shell
git show git-annex:remote.log
```

We can now activate the sibling with:

```shell
datalad siblings enable -s public-r2
```

We can now list all the activated siblings with:

```shell
datalad siblings
```

The result should show the `public-r2` sibling