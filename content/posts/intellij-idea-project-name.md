---
title: "Untangle Intellij Idea Project Names"
keywords: ["gradle", "intellij"]
tags: ["gradle", "idea", "intellij"]
date: 2021-04-22T12:42:30+02:00
---

Some releases ago idea started appending additional text in the project view to gradle sub-projects.
This has been more annoying than helpful so far. Especially in business related applications where
names tend to incorporate some conventions.

Here is a short view what idea does. The text in the squared brackets! There exists even an
[issue][0] that's been open for a long time. What's worse is that there is no option available in
the settings.

```
project
├───subprojects
|   ├───app [project.app]
│   └───library-x [project.library-x]
└─── // other stuff
```

Consider a name that is structured as follows : `<product>-<domain>-<subdomain>-<functional-area>`.
With some random business buzzwords for each part we end up with :`initech-contractmgmt-policy-bl`.
`bl` denotes here that it's a backend rather than a webapp.

```
initech-contractmgmt-policy-bl
├───subprojects
|   ├───application [initech-contractmgmt-policy-bl.application]
│   ├───infrastructure-api [initech-contractmgmt-policy-bl.infrastructure-api]
|   └───required-contractmgmt-partner-api [initech-contractmgmt-policy-bl.required-contractmgmt-partner-api]
└─── // other stuff
```

# Untangle the view

To get rid of those cluttered names in the project view:

* Open the menu `Help` and select `Edit Custom Properties`
* Create the file if it's missing
* Add the following property and save
  ```properties
  ide.hide.real.module.name=true
  ```
* Restart idea

After the restart the project view should look nice and clean.

```
project
├───subprojects
|   ├───app
│   └───library-x
└─── // other stuff
```

# Insight

Do you like the guide and want to give feedback or found a mistake? Then send me a mail
to `f4ntasyland /at/ protonmail /dot/ com`

You can always buy me a beer.
`（ ^_^）o自自o（^_^ ）`

_[xmr][0]:
473WTZ1gWFdjdEyioCQfbGQKRurZQoPDJLhuFJyubCrk4TRogKCtRum63bFMx2dP2y4AN1vf2fN6La7V7eB2cZ4vNJgMAcG_

[0]: https://youtrack.jetbrains.com/issue/IDEA-82965

[1]: https://www.getmonero.org/
