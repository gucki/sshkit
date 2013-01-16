# Usage Examples

## Run a command as a different user

    on hosts do |host|
      as 'www-data' do
        puts capture(:whoami)
      end
    end

## Run a command in a different directory

    on hosts do |host|
      within '/var/log' do
        puts capture(:head, '-n5', 'messages')
      end
    end

## Run a command with specific environmental variables

    on hosts do |host|
      with rack_env: :test do
        puts capture("env | grep RACK_ENV")
      end
    end

## Run a command in a different directory as a different user

    on hosts do |host|
      as 'www-data' do
        in '/var/log' do
          puts capture(:whoami)
          puts capture(:pwd)
        end
      end
    end

This will output:

    www-data
    /var/log

**Note:** This example is a bit misleading, as the `www-data` user doesn't
have a shell defined, one cannot switch to that user.

## Stack directory nestings:

    on hosts do
      in "/var" do
        puts capture(:pwd)
        in :log do
          puts capture(:pwd)
        end
      end
    end

This will output:

    /var/
    /var/log

The directory paths are joined using `File.join()`, which should correctly
join parts without forcing the user of the code to care about trailing or
leading slashes. It may be misleading as the `File.join()` is performed on the
machine running the code, if that's a Windows box, the paths may be incorrectly
joined according to the expectations of the machine receiving the commands.

## Do not care about the host block:

    on hosts do
      # The |host| argument is optional, it will
      # be nil in the block if not passed
    end

## Redirect all output to `/dev/null`

    SSHKit.config.output = File.open('/dev/null')

## Implement a dirt-simple formatter class:

    class MyFormatter < SSHKit::Formatter::Abstract
      def write(obj)
        case obj.is_a? SSHKit::Command
          # Do something here, see the SSHKit::Command documentation
        end
      end
    end

    SSHKit.config.output = MyFormatter.new($stdout)
    SSHKit.config.output = MyFormatter.new(SSHKit.config.output)
    SSHKit.config.output = MyFormatter.new(File.open('log/deploy.log', 'wb'))

## Set a password for a host.

    host = SSHKit::Host.new('user@example.com')
    host.password = "hackme"

    on host do |host|
      puts capture(:echo, "I don't care about security!")
    end

## Execute and raise an error if something goes wrong:

    on hosts do |host|
      execute!(:echo, '"Example Message!" 1>&2; false')
    end

This will raise `SSHKit::CommandUncleanExit.new("Example Message!")` which
will cause the command to abort.

## Do something different on one host, or another depending on a host property:

    host1 = SSHKit::Host.new 'user@example.com'
    host2 = SSHKit::Host.new 'user@example.org'

    on hosts do |host|
      target = "/var/www/sites/"
      if host.hostname =~ /org/
        target += "dotorg"
      else
        target += "dotcom"
      end
      execute! :git, :clone, "git@git.#{host.hostname}", target
    end

## Connect to a host in the easiest possible way:

    on 'example.com' do |host|
      execute :uptime
    end

This will resolve the `example.com` hostname into a `SSHKit::Host` object, and
try to pull up the correct configuration for it.


## Run a command without it being command-mapped:

If the command you attempt to call contains a space character it won't be
mapped:

    Command.new(:git, :push, :origin, :master).to_s
    # => /usr/bin/env git push origin master
    # (also: execute(:git, :push, :origin, :master)

    Command.new("git push origin master").to_s
    # => git push origin master
    # (also: execute("git push origin master"))

This can be used to access shell builtins (such as `if` and `test`)


## Run a command with a heredoc

An extension of the behaviour above, if you write a command like this:

    c = Command.new <<-EOCOMMAND
      if test -d /var/log
      then echo "Directory Exists"
      fi
    EOCOMMAND
    c.to_s
    # => if test -d /var/log; then echo "Directory Exists; fi
    # (also: execute <<- EOCOMMAND........))

**Note:** The logic which reformats the script into a oneliner may be naïve, but in all
known test cases, it works. The key thing is that `if` is not mapped to
`/usr/bin/env if`, which would break with a syntax error.