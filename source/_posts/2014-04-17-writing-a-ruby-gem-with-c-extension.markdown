---
layout: post
title: "Writing a Ruby gem with C extension"
date: 2014-04-17 17:37
comments: true
categories: [ruby, guide]
---
A couple weeks ago I sat down for a weekend and wrote my first Ruby gem with a C extension. [iStats](https://github.com/Chris911/iStats) is a command-line tool that allows you to easily grab the CPU temperature, fan speeds and battery information on OS X. The only way to access most of the metrics is via Apple's [IOKit](https://developer.apple.com/library/mac/documentation/devicedrivers/conceptual/IOKitFundamentals/Introduction/Introduction.html). The framework lets you access device drivers on machines running OS X and is written mostly in C. Therefos, I had to write the core of this gem in C but could still handle user input and the output in Ruby.

<!-- more -->

### Directory Structure
I always like to start a new gem using `bundle` to create the directory structure. Running `bundle gem my_gem` will get you a good starting directory structure. However, since we are writing a gem with a C extension we also want to add a folder for the C code. By convention, this folder is named `ext` and is placed at the root of the directory structure. This is the directory tree for my iStats gem:
```
iStats/
    Gemfile
    Gemfile.lock
    LICENSE
    README.md
    Rakefile
    bin
    └── istats
    ext
    └── osx_stats
        ├── extconf.rb
        ├── smc.c
        └── smc.h
    iStats.gemspec
    lib
    ├── iStats
    │   ├── command.rb
    │   ├── cpu.rb
    │   ├── [...]
    └── istats.rb
```

### C code and Makefile
As mentioned previously, the C code is found under the `ext` directory. Each C module should have its own subdirectory in there. You might also realize that there's another file in there named `extconf.rb`. This file is used when installing the gem to generate a Makefile that will be used to compile the C extension. Here's a basic example:
{% codeblock lang:ruby %}
require 'mkmf'

extension_name = 'osx_stats'

CONFIG['LDSHARED'] << ' -framework IOKit -framework CoreFoundation '

dir_config(extension_name)       # The destination
create_makefile(extension_name)  # Create Makefile
{% endcodeblock %}

Here's the [generated Makefile](https://gist.github.com/Chris911/6faa9e0bed37f96f9b0a). As this is the file that will be used to compile your code when someone installs the gem you should always make sure it compiles properly. Also keep in mind that the generated Makefile will be different depending on the Ruby version.

On top of your regular C code, you also need to map your C functions to Ruby modules and methods. To do this you need to include `ruby.h` and must have functions named `void Init_MODULE_NAME()`. This function is responsible for initializing the Ruby modules and defining your methods. You never call that function directly but it will be executed when you require the module. Here's what it would look like for the `get_cpu_temp` Ruby method that gets the CPU temperature as a C double and returns a Ruby float object.  
{% codeblock lang:c %}
VALUE CPU_STATS = Qnil; /* Ruby Module */

void Init_osx_stats() {
    CPU_STATS = rb_define_module("CPU_STATS");
    rb_define_method(CPU_STATS, "get_cpu_temp", method_get_cpu_temp, 0);
}

VALUE method_get_cpu_temp(VALUE self) {
    SMCOpen();
    double temp = SMCGetTemperature(SMC_KEY_CPU_TEMP);
    SMCClose();

    return rb_float_new(temp);
}
{% endcodeblock %}
Here's a great [cheat sheet for Ruby C extensions](http://blog.jacius.info/ruby-c-extension-cheat-sheet/) that shows the possible mappings between C types and Ruby objects. For a bigger example you can take a look at [the Ruby modules section](https://github.com/Chris911/iStats/blob/master/ext/osx_stats/smc.c#L313-L404) of my iStats gem.

### Gemfile
