# Ruby notes

use rbenv to manage ruby version

``` bash
brew install rbenv ruby-build
rbenv init

# list latest stable versions:
rbenv install -l

# list all local versions:
rbenv install -L

# install a Ruby version:
rbenv install 3.1.2

rbenv global 3.1.2   # set the default Ruby version for this machine
# or:
rbenv local 3.1.2    # set the Ruby version for this directory

# set the following in .zshrc
eval "$(rbenv init - zsh)"
```


## Add support of postgres

To use Postres DB, gem "pg" is required. 

Run the following if `bundle install` fails to work.
``` bash
brew install postgresql
```