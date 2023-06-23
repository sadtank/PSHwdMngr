# PSHwdMngr
Powershell CLI password manager

* FIPS 140-2 compliant encryption methods
* Fully in-memory, only needs this .ps1 and .bin (db file)
* PSH native (no modules or executable binaries needed)
* Import from KeePass
* Random password generation with granular complexity
* Diceware password generation with entropy calculation
* Quickly go to login url, automatic username/pwd copy to clipboard (and clear)

## First use
1. Run the .ps1, importing the functions into the current session. use `. .\PSHwdMngr.ps1` if current working directory and `. "<path>\PSHwdMngr.ps1"` for full path.
2. Ignoring all errors, either create a new entry or `Import-KeePassXML`.
3. Run `lse` to check the contents of `$global:db` in memory.
4. Run `savedb <path> -force $true` to save the DB at the desired directory. (Recommend script directory. Only adjust the $global:autoload variable in the script to match your new db path.)
5. Re-run one-liner from step 1 to clobber functions in memory to load your new auto-load path.
Your db should now auto load every time it's run and prompt for the decryption pwd.

I welcome feedback. This script is designed to work in less-permissive environments, where some .net namespaces/methods may not be available. Encryption relevant functions are grouped in the `Encryption SET` for easy code review reference. The use case for this script includes being able to transfer the db between computers. Many native credential objects entangle the user/computer, meaning the db could not be read on another machine. This instead uses a password as the AES key material, for symmetric encryption.

However, when the db is decrypted it sits in memory under a global variable which is a feature. From there you can access all PSH's cmdline goodness to manipulate the db. However, keep `Auto-Close` enabled and don't run anything else in that PSH window. Be aweare that any stdin and stdout may appear in logging, which could be centeralized in some enviornments. Db/key vars are cleared when the db is closed. Best practice is to exit the console and not trust the PSH garbage collector. And obviously, never leave your terminal unattended.

There are three base64 encoded strings. One is for the DiceWare word list and the other two are part of an easter egg. The paranoid among you will not trust anything I say to assure you they are benign. To inspect them without spoiling the fun, consider what cmdlets are calling the variable(s) holding these encoded strings. They are writing to the host and are not being invoked.

There is no builtin multilane text editor for PSH (or any Win CLI…). So to edit notes the user can choose to use CLI only or popup a form GUI. Check the `Edit-Notes` function (aliased to `vim`) to see the implementation. It’s easy to overwrite the notes field incorrectly, so I recommend using the GUI for multi-line notes whenever possible.

## Tips
- Recommend using `New-DiceWare` to generate a master password for the db. This will be memorable and quick to type. Consider using `New-DiceWare` when needing to generate a secure passphrase which is easy to remember/type (eg, host pwd, password db, etc). Use the random passowrd generator for passwords which can be auto typed by PSHwdMngr.
- Every change to your db is a "full send." It's not a bad idea to keep a backup of your db. It's not crazy to want to write down your db pwd somewhere safe... It's good enough for bitcoin millionaires who take custody of their bitcoin, keeping the seed phrase(s) on physical punch cards... which is considered best practice... just saying.
- Run `About-PSHwdMngr` for a list of functions. Most functions have an alias, (`List-Entries` = `lse`, `New-Entry` = `newe`). Most functions accept string and wildcard arguments thusly: `lse sharep*` (sharepoint) will return either a single complete record just like `Get-Entry` (`gete sharep*`) if there is a single match. Or it will return multiple matches if any. Any operation other than read will reject a query returning multiple entries.
- Use a NAS or cloud storage account to sync db updates to multiple machines in near-real time. Autosave will save every change to the db as soon as the change is confirmed. There is one edge case that could be improved. Where two users have the DB open, and both make a change, only the most recent change will survive. This could be fixed by updating the write functions to hold new changes in a record outside the db, each change would not only autosave but would quickly reopen the db, immediately append the temp record into the db, then quickly autosave. In this way the db would drastically reducing the chance of simultaneous edits by multiple users. However, where autoclose is used (default 5 mins), especially where users reopen the db explicitly before making edits, the current implementation is adequate.


## Common commands
```
#Load db: will also set $global:path
load-db <fullpath>

#Save db: defaults to $global:path
save-db

#Close db: will use autosave settings for save path
close-db

#Close db and exit the terminal (most secure)
done

#Force save, clobber any existing db at path with new db password
save-db <fullpath> -force $true 

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
