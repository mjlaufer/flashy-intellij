# Flashy IntelliJ Theme

[![Build](https://github.com/mjlaufer/flashy-intellij/workflows/Build/badge.svg)][gh:build]

<!-- Plugin description -->

Flashy is a dark theme for IntelliJ.

This is a port of [mjlaufer/flashy.nvim](https://www.github.com/mjlaufer/flashy.nvim).

> _Starting now, things are going to get real flashy!_ - Tengen Uzui

<!-- Plugin description end -->

### Table of contents

- [Gradle configuration](#gradle-configuration)
- [Predefined Run/Debug configurations](#predefined-rundebug-configurations)
- [Continuous integration](#continuous-integration) based on GitHub Actions
  - [Dependencies management](#dependencies-management) with Dependabot
  - [Changelog maintenance](#changelog-maintenance) with the Gradle Changelog Plugin
  - [Release flow](#release-flow) using GitHub Releases
  - [Plugin signing](#plugin-signing) with your private certificate
  - [Publishing the plugin](#publishing-the-plugin) with the Gradle IntelliJ Plugin

## Gradle configuration

The recommended method for plugin development involves using the [Gradle][gradle] setup with the [gradle-intellij-plugin][gh:gradle-intellij-plugin] installed.
The `gradle-intellij-plugin` makes it possible to run the IDE with your plugin and publish your plugin to JetBrains Marketplace.

> **Note**
>
> Make sure to always upgrade to the latest version of `gradle-intellij-plugin`.

This project includes a Gradle configuration already set up. This provides:

- Integration with the [gradle-intellij-plugin][gh:gradle-intellij-plugin] for smoother development.
- Integration with the [gradle-changelog-plugin][gh:gradle-changelog-plugin], which automatically patches the change notes based on the `CHANGELOG.md` file.
- [Plugin publishing][docs:publishing] using the token.

## Predefined Run/Debug configurations

Within the default project structure, there is a `.run` directory provided containing predefined *Run/Debug configurations* that expose corresponding Gradle tasks:

| Configuration name   | Description                                                                                                                                                                   |
|----------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Run Plugin           | Runs [`:runIde`][gh:gradle-intellij-plugin-runIde] Gradle IntelliJ Plugin task. Use the *Debug* icon for plugin debugging.                                                    |
| Run Verifications    | Runs [`:runPluginVerifier`][gh:gradle-intellij-plugin-runPluginVerifier] Gradle IntelliJ Plugin task to check the plugin compatibility against the specified IntelliJ IDEs.   |
| Run Tests            | Runs [`:test`][gradle:lifecycle-tasks] Gradle task.                                                                                                                           |
| Run IDE for UI Tests | Runs [`:runIdeForUiTests`][gh:intellij-ui-test-robot] Gradle IntelliJ Plugin task to allow for running UI tests within the IntelliJ IDE running instance.                     |
| Run Qodana           | Runs [`:runInspections`][gh:gradle-qodana-plugin] Gradle Qodana Plugin task. Starts Qodana inspections in a Docker container and serves generated report on `localhost:8080`. |

> **Note**
>
> You can find the logs from the running task in the `idea.log` tab.

## Continuous integration

Continuous integration depends on [GitHub Actions][gh:actions], a set of workflows that make it possible to automate your testing and release process.
Thanks to such automation, you can delegate the testing and verification phases to the Continuous Integration (CI) and instead focus on development (and writing more tests).

In the `.github/workflows` directory, you can find definitions for the following GitHub Actions workflows:

- [Build](.github/workflows/build.yml)
  - Triggered on `push` and `pull_request` events.
  - Runs the *Gradle Wrapper Validation Action* to verify the wrapper's checksum.
  - Runs the `verifyPlugin` and `test` Gradle tasks.
  - Builds the plugin with the `buildPlugin` Gradle task and provides the artifact for the next jobs in the workflow.
  - Verifies the plugin using the *IntelliJ Plugin Verifier* tool.
  - Prepares a draft release of the GitHub Releases page for manual verification.
- [Release](.github/workflows/release.yml)
  - Triggered on `released` event.
  - Updates `CHANGELOG.md` file with the content provided with the release note.
  - Signs the plugin with a provided certificate before publishing.
  - Publishes the plugin to JetBrains Marketplace using the provided `PUBLISH_TOKEN`.
  - Sets publish channel depending on the plugin version, i.e. `1.0.0-beta` -> `beta` channel.
  - Patches the Changelog and commits.
- [Run UI Tests](.github/workflows/run-ui-tests.yml)
  - Triggered manually.
  - Runs for macOS, Windows, and Linux separately.
  - Runs `runIdeForUiTests` and `test` Gradle tasks.

All the workflow files have accurate documentation, so it's a good idea to take a look through their sources.

### Dependencies management

This Template project depends on Gradle plugins and external libraries – and during the development, you will add more of them.

All plugins and dependencies used by Gradle are managed with [Gradle version catalog][gradle:version-catalog], which defines versions and coordinates of your dependencies in the [`gradle/libs.versions.toml`][file:libs.versions.toml] file.

> **Note**
>
> To add a new dependency to the project, in the `dependencies { ... }` block, add:
>
> ```kotlin
> dependencies {
>   implementation(libs.annotations)
> }
> ```
>
> and define the dependency in the [`gradle/libs.versions.toml`][file:libs.versions.toml] file as follows:
> ```toml
> [versions]
> annotations = "24.1.0"
>
> [libraries]
> annotations = { group = "org.jetbrains", name = "annotations", version.ref = "annotations" }
> ```

Keeping the project in good shape and having all the dependencies up-to-date requires time and effort, but it is possible to automate that process using [Dependabot][gh:dependabot].

Dependabot is a bot provided by GitHub to check the build configuration files and review any outdated or insecure dependencies of yours – in case if any update is available, it creates a new pull request providing the proper change.

> **Note**
>
> Dependabot doesn't yet support checking of the Gradle Wrapper.
> Check the [Gradle Releases][gradle:releases] page and update your `gradle.properties` file with:
> ```properties
> gradleVersion = ...
> ```
> and run
> ```bash
> ./gradlew wrapper
> ```

### Changelog maintenance

When releasing an update, it is essential to let your users know what the new version offers.
The best way to do this is to provide release notes.

The changelog is a curated list that contains information about any new features, fixes, and deprecations.
When they're provided, these lists are available in a few different places:
- the [CHANGELOG.md](./CHANGELOG.md) file,
- the [Releases page][gh:releases],
- the *What's new* section of JetBrains Marketplace Plugin page,
- and inside the Plugin Manager's item details.

There are many methods for handling the project's changelog.
The one used in the current template project is the [Keep a Changelog][keep-a-changelog] approach.

The [Gradle Changelog Plugin][gh:gradle-changelog-plugin] takes care of propagating information provided within the [CHANGELOG.md](./CHANGELOG.md) to the [Gradle IntelliJ Plugin][gh:gradle-intellij-plugin].
You only have to take care of writing down the actual changes in proper sections of the `[Unreleased]` section.

When releasing a plugin update, you don't have to care about bumping the `[Unreleased]` header to the upcoming version – it will be handled automatically on the Continuous Integration (CI) after you publish your plugin.
GitHub Actions will swap it and provide you an empty section for the next release so that you can proceed with your development.

### Release flow

The release process depends on the workflows already described above.
When your main branch receives a new pull request or a direct push, the [Build](.github/workflows/build.yml) workflow runs multiple tests on your plugin and prepares a draft release.

The draft release is a working copy of a release, which you can review before publishing.
It includes a predefined title and git tag, the current plugin version, for example, `v0.0.1`.
The changelog is provided automatically using the [gradle-changelog-plugin][gh:gradle-changelog-plugin].
An artifact file is also built with the plugin attached.
Every new Build overrides the previous draft to keep your *Releases* page clean.

When you edit the draft and use the <kbd>Publish release</kbd> button, GitHub will tag your repository with the given version and add a new entry to the Releases tab.
Next, it will notify users who are *watching* the repository, triggering the final [Release](.github/workflows/release.yml) workflow.

### Plugin signing

Plugin Signing is a mechanism introduced in the 2021.2 release cycle to increase security in [JetBrains Marketplace](https://plugins.jetbrains.com) and all of our IntelliJ-based IDEs.

JetBrains Marketplace signing is designed to ensure that plugins aren't modified over the course of the publishing and delivery pipeline.

The current project provides a predefined plugin signing configuration that lets you sign and publish your plugin from the Continuous Integration (CI) and local environments.
All the configuration related to the signing should be provided using [environment variables](#environment-variables).

To find out how to generate signing certificates, check the [Plugin Signing][docs:plugin-signing] section in the IntelliJ Platform Plugin SDK documentation.

> **Note**
>
> Remember to encode your secret environment variables using `base64` encoding to avoid issues with multi-line values.

### Publishing the plugin

Releasing a plugin to JetBrains Marketplace is a straightforward operation that uses the `publishPlugin` Gradle task provided by the [gradle-intellij-plugin][gh:gradle-intellij-plugin-docs].
In addition, the [Release](.github/workflows/release.yml) workflow automates this process by running the task when a new release appears in the GitHub Releases section.

> **Note**
>
> Set a suffix to the plugin version to publish it in the custom repository channel, i.e. `v1.0.0-beta` will push your plugin to the `beta` [release channel][docs:release-channel].

The authorization process relies on the `PUBLISH_TOKEN` secret environment variable, specified in the _Secrets_ section of the repository _Settings_.

You can get that token in your JetBrains Marketplace profile dashboard in the [My Tokens][jb:my-tokens] tab.

> **Warning**
>
> Before using the automated deployment process, it is necessary to manually create a new plugin in JetBrains Marketplace to specify options like the license, repository URL, etc.
> Please follow the [Publishing a Plugin][docs:publishing] instructions.

[docs:publishing]: https://plugins.jetbrains.com/docs/intellij/publishing-plugin.html?from=IJPluginTemplate
[docs:release-channel]: https://plugins.jetbrains.com/docs/intellij/publishing-plugin.html?from=IJPluginTemplate#specifying-a-release-channel
[docs:plugin-signing]: https://plugins.jetbrains.com/docs/intellij/plugin-signing.html?from=IJPluginTemplate

[file:libs.versions.toml]: ./gradle/libs.versions.toml

[gh:actions]: https://help.github.com/en/actions
[gh:build]: https://github.com/mjlaufer/flashy-intellij/actions?query=workflow%3ABuild
[gh:dependabot]: https://docs.github.com/en/free-pro-team@latest/github/administering-a-repository/keeping-your-dependencies-updated-automatically
[gh:gradle-changelog-plugin]: https://github.com/JetBrains/gradle-changelog-plugin
[gh:gradle-intellij-plugin]: https://github.com/JetBrains/gradle-intellij-plugin
[gh:gradle-intellij-plugin-docs]: https://plugins.jetbrains.com/docs/intellij/tools-gradle-intellij-plugin.html
[gh:gradle-intellij-plugin-runIde]: https://plugins.jetbrains.com/docs/intellij/tools-gradle-intellij-plugin.html#tasks-runide
[gh:gradle-intellij-plugin-runPluginVerifier]: https://plugins.jetbrains.com/docs/intellij/tools-gradle-intellij-plugin.html#tasks-runpluginverifier
[gh:gradle-qodana-plugin]: https://github.com/JetBrains/gradle-qodana-plugin
[gh:intellij-ui-test-robot]: https://github.com/JetBrains/intellij-ui-test-robot
[gh:releases]: https://github.com/mjlaufer/flashy-intellij/releases

[gradle]: https://gradle.org
[gradle:build-cache]: https://docs.gradle.org/current/userguide/build_cache.html
[gradle:configuration-cache]: https://docs.gradle.org/current/userguide/configuration_cache.html
[gradle:lifecycle-tasks]: https://docs.gradle.org/current/userguide/java_plugin.html#lifecycle_tasks
[gradle:releases]: https://gradle.org/releases
[gradle:version-catalog]: https://docs.gradle.org/current/userguide/platforms.html#sub:version-catalog

[jb:my-tokens]: https://plugins.jetbrains.com/author/me/tokens

[keep-a-changelog]: https://keepachangelog.com
[keep-a-changelog-how]: https://keepachangelog.com/en/1.0.0/#how
