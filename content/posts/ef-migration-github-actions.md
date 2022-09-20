+++ 
draft = false
date = 2022-08-26T09:00:00+10:00
title = "Entity Framework Migrations from GitHub Actions"
description = ""
slug = ""
authors = []
tags = ['entity-framework', 'github-actions']
categories = []
externalLink = ""
series = []
+++

So you have an app using Entity Framework that you're deploying using GitHub Actions, and you want to use Migrations to keep your database up to date.

<!--more-->

But there might be several reasons why you don't want your app to apply the migrations (as described in the [Apply Migrations at Runtime](https://docs.microsoft.com/en-us/ef/core/managing-schemas/migrations/applying?tabs=dotnet-core-cli#apply-migrations-at-runtime) section of the documentation). My two favourite reasons are:

* You only want your application to have db_datareader/db_datawriter access to your database, so it can't make schema changes
* Your app might be running in multiple places and you don't want to risk two instances both trying to apply a migration concurrently

And honestly, it just doesn't *feel right* to have the app in charge of the database schema. The *deployment* mechanism should be the thing ensuring that dependencies, of which the database is one, are all correctly set up.

With that in mind it makes sense to apply database migrations from your GitHub Actions workflow, if that's what you're using to deploy your app. So let's do that!

We'll assume that you already have a GitHub Action to build and deploy your app.

## Install EF Tool

The first thing you'll need to do in your GitHub action, after the "Set up .NET" step, is to install the Entity Framework command line tool:

```yaml
- name: Install EF Tool
    run: |
        dotnet new tool-manifest
        dotnet tool install dotnet-ef
```

So now we have the `dotnet ef migrations` command at our disposal.

## Generate Migration Bundle

Next up, after your build/publish step, you'll want to generate the migration. There are several ways to do this, but I'm using "bundles", which generates an exe that can apply the migration:

```yaml
- name: Generate EF migration script
    run: dotnet ef migrations bundle --startup-project MyApp --output ${{env.DOTNET_ROOT}}/myapp/efbundle.exe --configuration Bundle
```

{{< notice note >}}
You'll notice the `--configuration Bundle` on the end there. That's a hack to work around [this issue](https://github.com/dotnet/efcore/issues/25555), which I really hope is fixed in the EF 7 release. What you *should* be able to do, since you just finished building the solution, is pass `--configuration Release` and `--no-build` so it uses the already-built binaries to create the bundle.
{{< /notice >}}

If you're more SQL-script oriented, you might instead like to generate an idempotent script, which you can do like this:

```yaml
- name: Generate EF migration script
    run: dotnet ef migrations script --idempotent --startup-project MyApp --output ${{env.DOTNET_ROOT}}/myapp/migrate.sql
```

In both cases, we're dropping the bundle (be it an exe or an SQL script) into the `${{env.DOTNET_ROOT}}/myapp` folder, so it will be part of the deployable artifact of our app. That's useful if you have separate "build" and "deploy" jobs in your yaml file.

We'll talk about the different ways to apply these migrations next.

## Apply the Migration

Now that your migration is built and ready to apply, let's apply it! 

You'll need a GitHub secret, which I've called `MY_CONNECTION_STRING` in these examples, to tell the action how to connect to your database. Remember that the credentials in that connection string will need enough access to your database to make schema changes!

For a bundle executable, this step will look like this:

```yaml
  - name: Apply EF migration script
        run: ./efbundle.exe --connection "${{ secrets.MY_CONNECTION_STRING }}"
        # or ./myapp/efbundle.exe if you're in the same job as the build
```

So that just runs the `efbundle.exe` command that we generated in a previous step, and passes it that secret connection string. 

For an idempotent SQL script, the step will look like this:

```yaml
  - name: Apply EF migration script
        uses: Azure/sql-action@v1.2
        with:
          connection-string: ${{ secrets.MY_CONNECTION_STRING }}
          sql-file: ./migrate.sql
          # or ./myapp/migrate.sql if you're in the same job as the build
```

So this is using the Microsoft-maintained `Azure/sql-action` step, passing it our connection string and the script we created earlier. I've not been able to use this technique yet, because the db_owner user in my connection string only exists in my app's database, but the sql-action step requires access to the master database on your server. That will change in v2 of the action, due out soon.

You'll want to have this step happen before the actual deployment of your app, so you're not deploying the app if something goes wrong with the migration.

And there you have it! Create your migration and apply it within your GitHub Actions workflow! 