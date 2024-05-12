---
title: "Smart Directory Navigation with PowerShell Script"
date: 2024-05-12T11:09:42+08:00
draft: false
author: "Yu-Kai \"Steven\" Wang"
tags: [powershell, scripting, automation, CLI]
categories: [devops]
featuredImage: '/images/futuristic_dashboard.png'
---

Sometimes I find myself having to navigate between multiple projects. And since my main working laptop is on *Windows*, 
naturally I tried to customize my command line tool (I use *PowerShell* because *WSL* is just way too slow) to jump between directories faster. My folder organization looks something like this:

```
home
│
├───Desktop
│   │   file011.txt
│   │   file012.txt
│   │
│   └───projects
│       ├───project1
│       │   │   file011.txt
│       │   │   ...
│       │
│       ├───project2
│       │   │   file011.txt
│       │   │   ...
│       │
│       ├───project3
│       │   │   file011.txt
│       │   │   ...
│       │
│       │   ...
│   ...  
```

My goal is to be able to easily switch between project folders (in the terminal), no matter where my current directory points to. For example, I could be under `/Downloads`, and want to switch to `/Desktop/projects/project3`. Instead of scratching my head trying to type out the entire absolute or relative path, I thought I could use the help of some custom PowerShell script.

{{< figure src="./images/programmers_automation_meme.jpg" caption="It do be like that sometimes" >}}

### Scripting with PowerShell

The plan is to create a custom command `projects <project-name>` with tab auto-completion to change the directory to that project from anywhere. This is surprisingly easy with PowerShell.

```powershell
# ValidateSet, find the exhaustive list of the project folders for auto-completion
Class MyProjects: System.Management.Automation.IValidateSetValuesGenerator {
    [string[]] GetValidValues() {
        $ProjectPaths = "C:/Users/$ENV:USERNAME/Desktop/projects/"
        $ProjectNames = ForEach ($ProjectPath in $ProjectPaths) {
            If (Test-Path $ProjectPath) {
                (Get-ChildItem $ProjectPath).BaseName
            }
        }
        return [string[]] $ProjectNames
    }
}

# Command Definition
function Enter-Projects {
    # cd to subfolders under "/projects"
    # https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions_argument_completion?view=powershell-7.2
    [CmdletBinding()]
    param(
        [ValidateSet([MyProjects])]
        [ArgumentCompletions([MyProjects])]
        [string]
        $projects
    )
    Set-Location -Path "C:/Users/$ENV:USERNAME/Desktop/projects/$projects" &&
    Get-ChildItem
}

# Alias
New-Alias -Name "projects" Enter-Projects
```

{{< admonition type=note title="" open=true >}}
I am using the latest release of PowerShell 7.4.2, the default PowerShell 5 that came with Windows 10 (the blue one) has a slightly different syntax.
{{< /admonition >}}

We'll go through the script step by step.

In lines 1 ~ 12 we define the class `MyProject`. This class is a `ValidateSet`, which defines all the valid arguments to our custom command for auto-completion. In this case, it is just an exhaustive list of all the project names. At lines 5 ~ 9, we use a for-each loop to get all the project names under our base project folder and return them as a list of strings at line 10. See the [official PowerShell docs](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions_argument_completion?view=powershell-7.2#dynamic-validateset-values-using-classes) for more info.

From lines 15 ~ 27, we define the actual command functionality. I named the function `Enter-Projects` to follow the PowerShell commands' [naming convention](https://learn.microsoft.com/en-us/powershell/scripting/developer/cmdlet/approved-verbs-for-windows-powershell-commands?view=powershell-7.4). Line 18 we call the `CmdletBinding()` to declare this function as an advanced function to use our custom-defined arguments.

Line 19 ~ 24 we load our `MyProjects` validate set as the valid arguments for the command's auto-completion.

Lines 25 ~ 26 are the body of the function. We first call `Set-Location`, which is just the full name of the `cd` command, to change our current directory to the specified parameter `$projects`. Next, we call `Get-Children` (full name of `ls`) to list out all the items under the destination project's folder. 

Finally, at line 30, we create an *alias* "`projects`" for our `Enter-Projects` function, because no one wants to type out the full name of the function.

### Deployment

{{< figure src="./images/terminal.JPG" caption="\"study\" command I made using the same way to navigate school related folders" >}}

Save the file as `/Documents/PowerShell/Microsoft.PowerShell_profile.ps1` and reopen your terminal so that the script is loaded. 
You can now type in `projects` in your terminal and press tab to go through your list of projects. Pressing enter will change your current directory to the project folder and list out all the items in it.

That's it!

### References
[Functions Argument Completion](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions_argument_completion?view=powershell-7.2)
