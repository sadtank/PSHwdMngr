# PSHwdMngr
Powershell CLI password manager

* FIPS 140-2 compliant encryption methods
* Fully in-memory, only needs this .ps1 and .bin (db file)
* PSH native (no modules or executable binaries needed)
* Import from KeePass
* Random password generation with granular complexity
* Diceware password generation with entropy calculation
* Quickly go to login url, automatic username/pwd copy to clipboard (and clear)

Run `About-PSHwdMngr` for a list of functions. Most functions have an alias, (`List-Entries` = `lse`, `New-Entry` = `newe`). Most functions accept string arguments thusly: `lse sharep*` (sharepoint) will return either a single complete record just like `Get-Entry` (`gete sharep*`) if there is a single match. Or it will return multiple matches if there are any.

The encryption method on disk appears to be proper. However, I welcome feedback. This script is designed to work in less-permissive environments, where some .net namespaces/methods may not be available. Encryption relevant functions are grouped in the `Encryption SET` for easy code review reference. The use case for this script includes being able to transfer the db between computers. Many native credential objects entangle the user/computer, meaning the db could not be read on another machine. This instead uses a password as the AES key material, for symmetric encryption.

However, when the db is decrypted it sits in memory under a global variable, and that’s a feature. From there you can access all PSH's cmdline goodness to manipulate the db. However, keep `Auto-Close` enabled and don't run anything else in that PSH window. db/key vars are cleared when the db is closed. Close the window so as not to trust the PSH garbage collector. Never leave your terminal unattended.

Recommend using `New-DiceWare` to generate a master password for the db. This will be memorable and quick to type. It's not a bad idea to keep a backup of your db.

There are three base64 encoded strings. One is for the DiceWare word list and the other two are part of an easter egg. The paranoid among you will want to inspect those, but try not to spoil the fun.

There is no builtin multilane text editor for PSH (or any Win CLI…). So to edit notes the user can choose to use CLI only or popup a form GUI. Check the `Edit-Notes` function (aliased to `vim`) to see the implementation. It’s easy to overwrite the notes field incorrectly, so I recommend using the GUI for multi-line notes whenever possible.

Common commands:
```
#Load db: will also set $global:path
load-db <fullpath>

#Save db: defaults to $global:path
#save-db

#Force save, clobber with new db password
#save-db <fullpath> -force $true 
#new entry
newe

#edit entry
edite

#remove entry
#if rm is aliased to Remove-Item, this may be typo prone as it could run against the current working directory...
rme “git h*”

#get entries, accepts wildcard. Use quotes for spaces
lse “git h*”

#use entry
usee “git h*”

#use pwd
usepwd “git h*”

#go to url
useurl “git h*”

#new pwd
Get-RandomPassword

#new diceword generation
New-DiceWare <number of words>
```
