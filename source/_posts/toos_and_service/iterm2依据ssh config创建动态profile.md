---
title: iterm2依据ssh config创建动态profile
date: 2019-06-07 12:12:28
tags:
- mac
---

通过创建ssh config文件来创建iterm2的profile，然后达到类似于xhell的文件标签效果

<!--more-->

# 准备ssh config

创建文件 ~/.ssh/config

```bash
Host tx.k8s.config
    HostName host-ip
    User root
    IdentityFile ~/.ssh/id_rsa
```

# 准备工具

## brew安装

```bash
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```
## 安装perl json插件

```bash
brew install cpanm
sudo cpanm install JSON
```

# 准备perl脚本

[脚本地址](https://gist.github.com/rsperl/bcc7fbb845b2cf5c260a)

```perl
#!/usr/bin/env perl

# 
# licensed under GPL v2, same as iTerm2 https://www.iterm2.com/license.txt
#

use strict;
use JSON;

my $output = $ENV{HOME} . "/Library/Application\ Support/iTerm2/DynamicProfiles/profiles.json";
print STDERR "profiles will be written to '$output'\n";

my @profiles;
my $ssh_config = $ENV{HOME} . "/.ssh/config";
open my $fh, '<', $ssh_config;
my @contents = <$fh>;
close $fh;

my $name;
my $custom_command;
my @tags;
for(my $i=0; $i<@contents; $i++) {
    my $line = $contents[$i];
    chomp $line;
    next if index($line, '*') >= 0;
    if( $line =~ /^Host\s+(.+?)$/ ) {
        my $match = $1;
        if( $match =~ /(.+?)\s+#\s+Tags=(.+?)$/ ) {
            $name = $1;
            @tags = split(/\s*,\s*/, $2);
        } else {
            $name = $match;
            @tags = ();
        }
    } elsif( $line =~ /^\s*$/ ) {
        next unless $name;
        print STDERR "adding profile for $name\n";
        $custom_command = "ssh $name";
        my $p = {
            Name => $name,
            Guid => $name,
            Shortcut => "",
            "Custom Command" => "Yes",
            Command => $custom_command,
        };
        $p->{Tags} = [@tags] if( @tags );
        $name           = "";
        $custom_command = "";
        @tags           = ();
        push(@profiles, $p);
    };
}

my $json =  to_json({ Profiles => \@profiles }, { pretty => 1 });
open my $fh, '>', $output;
print $fh $json;
close $fh;
print STDERR "finished\n";
```

# 直接执行

```bash
./generate_iterm2_dynamic_profiles.pl
```

# 使用

打开iterm2 使用快捷键 command + o，既可打开profile文件页面