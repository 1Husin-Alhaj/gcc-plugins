Install gcc
===========

0. Make sure you fulfill all the gcc dependences and set an installation path

    $ sudo apt-get install libgmp-dev libmpc-dev libmpfr-dev
    $ export INSTALLDIR=$HOME/gcc/gcc-install

1. Download gcc

    $ wget http://ftp.gnu.org/gnu/gcc/gcc-4.8.2/gcc-4.8.2.tar.bz2

2. Extract

    $ tar xfj gcc-4.8.2.tar.bz2

3. Create a build directory

    $ mkdir gcc-build
    $ cd gcc-build

4. Configure to get a C/C++ compiler

   $ ../gcc-4.8.2/configure --prefix=$INSTALLDIR --enable-languages=c,c++

5. Build (will take a while, like 10 min or so in a fast computer)

   $ make -j$(getconf _NPROCESSORS_ONLN) 

6. Install

   $ make install

7. Ensure we have plugins

   $ ${INSTALLDIR}/bin/gcc -print-file-name=plugin
   <<INSTALLDIR>>/lib/gcc/x86_64-unknown-linux-gnu/4.8.2/plugin

If it appears 'plugin' you are using the wrong compiler


Our first plugin
================

Create a boilerplate Makefile.common

    # Common makefile to be included from all other makefiles
    
    # Where we installed gcc and its headers
    INSTALLDIR=<<INSTALLDIR>>
    
    CC=$(INSTALLDIR)/bin/gcc
    PLUGINDIR=$(shell $(CC) -print-file-name=plugin)
    
    CFLAGS=-fPIC -Wall -I$(PLUGINDIR)/include
    LDFLAGS=
    LDADD=
    
    END=
    OBJECTS=$(patsubst %.c,%.o,$(SOURCES))
    
    all: $(PLUGIN)
    
    $(PLUGIN): $(OBJECTS)
    	$(CC) $(LDFLAGS) -o $@ -shared $+ $(LDADD)
    
    
    .PHONY: all clean
    clean:
    	rm -f $(OBJECTS) $(PLUGIN)

Create a first/Makefile

    PLUGIN=my-first-gcc-plugin.so
    SOURCES=\
            my-first-gcc-plugin.c \
    		$(END)
    
    include ../Makefile.common
  

Create a phase that just prints what it is passed by gcc


    // This is the first gcc header to be included
    #include "gcc-plugin.h"
    #include "plugin-version.h"
    
    #include <stdio.h>
    
    // We must assert that this plugin is GPL compatible
    int plugin_is_GPL_compatible;
    
    	      
    int plugin_init (struct plugin_name_args *plugin_info,
    		struct plugin_gcc_version *version)
    {
    	// We check the current gcc loading this plugin against the gcc we used to
    	// created this plugin
    	if (!plugin_default_version_check (version, &gcc_version))
        {
            fprintf(stderr, "This GCC plugin is for version %d.%d\n", GCCPLUGIN_VERSION_MAJOR, GCCPLUGIN_VERSION_MINOR);
    		return 1;
        }
    
        // Let's print all the information given to this plugin!
    
        fprintf(stderr, "Plugin info\n");
        fprintf(stderr, "===========\n\n");
        fprintf(stderr, "Base name: %s\n", plugin_info->base_name);
        fprintf(stderr, "Full name: %s\n", plugin_info->full_name);
        fprintf(stderr, "Number of arguments of this plugin: %d\n", plugin_info->argc);
        int i;
        for (i = 0; i < plugin_info->argc; i++)
        {
            fprintf(stderr, "Argument %d: Key: %s. Value: %s\n", i, plugin_info->argv[i].key, plugin_info->argv[i].value);
    
        }
        fprintf(stderr, "Version string of the plugin: %s\n", plugin_info->version);
        fprintf(stderr, "Help string of the plugin: %s\n", plugin_info->help);
    
        fprintf(stderr, "\n");
        fprintf(stderr, "Version info\n");
        fprintf(stderr, "============\n\n");
        fprintf(stderr, "Base version: %s\n", version->basever);
        fprintf(stderr, "Date stamp: %s\n", version->datestamp);
        fprintf(stderr, "Dev phase: %s\n", version->devphase);
        fprintf(stderr, "Revision: %s\n", version->devphase);
        fprintf(stderr, "Configuration arguments: %s\n", version->configuration_arguments);
        fprintf(stderr, "\n");
    
        fprintf(stderr, "Plugin successfully initialized\n");
    
        return 0;
    }

Compile

    $ make
    <<INSTALLDIR>>/bin/gcc -fPIC -Wall -I<<INSTALLDIR>>/lib/gcc/x86_64-unknown-linux-gnu/4.8.2/plugin/include   -c -o my-first-gcc-plugin.o my-first-gcc-plugin.c
    <<INSTALLDIR>>/bin/gcc  -o my-first-gcc-plugin.so -shared my-first-gcc-plugin.o 

Run gcc enabling the plugin

    $ <<INSTALLDIR>>/bin/gcc -fplugin=./my-first-gcc-plugin.so -c test.c
    
    Plugin info
    ===========
    
    Base name: my-first-gcc-plugin
    Full name: /home/roger/soft/gcc-plugins/first/my-first-gcc-plugin.so
    Number of arguments of this plugin: 0
    Version string of the plugin: (null)
    Help string of the plugin: (null)
    
    Version info
    ============
    
    Base version: 4.8.2
    Date stamp: 20131016
    Dev phase: 
    Revision: 
    Configuration arguments: ../gcc-4.8.2/configure --prefix=<<INSTALLDIR>> --enable-languages=c,c++
    
    Plugin successfully initialized

The phase does nothing and still lacks info but it is a step in the good direction.

Introducing ourselves to the compiler
=====================================
  
    // This is the first gcc header to be included
    #include "gcc-plugin.h"
    #include "plugin-version.h"
    
    #include <stdio.h>
    
    // We must assert that this plugin is GPL compatible
    int plugin_is_GPL_compatible;
    
    static struct plugin_info my_gcc_plugin_info = { "1.0", "This is a very simple plugin" };
    
    int plugin_init (struct plugin_name_args *plugin_info,
    		struct plugin_gcc_version *version)
    {
    	// We check the current gcc loading this plugin against the gcc we used to
    	// created this plugin
    	if (!plugin_default_version_check (version, &gcc_version))
        {
            fprintf(stderr, "This GCC plugin is for version %d.%d\n", GCCPLUGIN_VERSION_MAJOR, GCCPLUGIN_VERSION_MINOR);
    		return 1;
        }
    
        register_callback(plugin_info->base_name,
                /* event */ PLUGIN_INFO,
                /* callback */ NULL, /* user_data */ &my_gcc_plugin_info);
    
        return 0;
    }


Call the C compiler (the gcc driver is not going to be enough) to see the plugins loaded

    $ <<INSTALLDIR>>/libexec/gcc/x86_64-unknown-linux-gnu/4.8.2/cc1 -fplugin=./help-version.so --help
    
    ...
      -p                                [disabled]
      -pedantic-errors                  [disabled]
      -quiet                            [disabled]
    
    Help for the loaded plugins:
     help-version:
        This is a very simple plugin

 Being able to pass parameters
===========================================================

Once we have introduced ourselves to the compiler we can pass parameters using -fplugin-arg-<<basename>>-key=value


    // This is the first gcc header to be included
    #include "gcc-plugin.h"
    #include "plugin-version.h"
    
    #include <stdio.h>
    
    // We must assert that this plugin is GPL compatible
    int plugin_is_GPL_compatible;
    
    static struct plugin_info my_gcc_plugin_info = { "1.0", "This is a very simple plugin" };
    
    int plugin_init (struct plugin_name_args *plugin_info,
    		struct plugin_gcc_version *version)
    {
    	// We check the current gcc loading this plugin against the gcc we used to
    	// created this plugin
    	if (!plugin_default_version_check (version, &gcc_version))
        {
            fprintf(stderr, "This GCC plugin is for version %d.%d\n", GCCPLUGIN_VERSION_MAJOR, GCCPLUGIN_VERSION_MINOR);
    		return 1;
        }
    
        register_callback(plugin_info->base_name,
                /* event */ PLUGIN_INFO,
                /* callback */ NULL, /* user_data */ &my_gcc_plugin_info);
    
        fprintf(stderr, "Number of arguments of this plugin: %d\n", plugin_info->argc);
        int i;
        for (i = 0; i < plugin_info->argc; i++)
        {
            fprintf(stderr, "Argument %d: Key: \"%s\" Value: \"%s\"\n", i, plugin_info->argv[i].key, plugin_info->argv[i].value);
        }
    
        return 0;
    }


(To be continued...)
