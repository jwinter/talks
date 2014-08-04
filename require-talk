# first slide

Use me to make sure that the rest are gonna look ok

# require 'require'

What happens when we require

# require is a method

irb(main):002:0> method :require
=> #<Method: Object(Kernel)#require>

"Object(Kernel)#require" means that:
- require is defined in the Kernel module 
- and mixed into Ruby's base Object class

# where does require live?

2.0.0-p353 :003 > require_method = Kernel.method(:require)
 => #<Method: Kernel.require> 
2.0.0-p353 :005 > require_method.source_location
 => nil 

# sad

:(

#

Let's find the source anyway

# This is what it looks like in JRuby: core/src/main/java/org/jruby/RubyKernel.java

```java
    @JRubyMethod(name = "require", module = true, visibility = PRIVATE)
    public static IRubyObject require19(ThreadContext context, IRubyObject recv, IRubyObject name, Block block) {
        // HEYJOE: ^ Threadcontext arg is necessary, name is the arg to require, block ain't used
        Ruby runtime = context.runtime; //HEYJOE: runtime is like a whole Ruby VM
        IRubyObject tmp = name.checkStringType();

        //Normal happy path, name isn't nil
        if (!tmp.isNil()) return requireCommon(runtime, recv, tmp, block);

        //handles to_path
        return requireCommon(runtime, recv, RubyFile.get_path(context, name), block);
    }

    private static IRubyObject requireCommon(Ruby runtime, IRubyObject recv, IRubyObject name, Block block) {
        return runtime.newBoolean(runtime.getLoadService().require(name.convertToString().toString()));
    }
```

# and in Rubinius: kernel/common/code_loader.rb

```rb
    # Searches for and loads Ruby source files and shared library extension
    # files. See CodeLoader.require for the rest of Kernel#require
#NB: this is CodeLoader.require o_O
    # functionality.
    def require
      wait = false
      req = nil

#NB Map of already laoded files
      reqs = CodeLoader.load_map

#NB Thread-related stuff here
      Rubinius.synchronize(Lock) do
        return unless resolve_require_path

        if req = reqs[@load_path]
          return if req.current_thread?
          wait = true
        else
          req = RequireRequest.new(@type, reqs, @load_path)
# RequireRequest is does some of the thread sync abstraction
          reqs[@load_path] = req
          req.take!
        end
      end
```

# moar requiring

```rb
      if wait
        if req.wait
          # While waiting the code was loaded by another thread.
          # We need to release the lock so other threads can continue too.
          req.unlock
          return false
        end

        # The other thread doing the lock raised an exception
        # through the require, so we try and load it again here.
      end

      if @type == :ruby
        load_file
      else
        load_library
      end

      return req
    end
```

# and in MRI (ruby.c):

```c
static void
require_libraries(VALUE *req_list)
{
    VALUE list = *req_list;
    VALUE self = rb_vm_top_self();
    ID require;
    rb_thread_t *th = GET_THREAD();
    rb_encoding *extenc = rb_default_external_encoding();
    int prev_parse_in_eval = th->parse_in_eval;
    th->parse_in_eval = 0;

    CONST_ID(require, "require");
    while (list && RARRAY_LEN(list) > 0) {
	VALUE feature = rb_ary_shift(list);
	rb_enc_associate(feature, extenc);
	RBASIC_SET_CLASS_RAW(feature, rb_cString);
	OBJ_FREEZE(feature);
	rb_funcall2(self, require, 1, &feature);
    }
    *req_list = 0;

    th->parse_in_eval = prev_parse_in_eval;
}
```

# Also found this in the source, line 1704 in ruby.c, thought it was funny:

```c
        ruby_set_script_name(opt->script_name);
	require_libraries(&opt->req_list);	/* Why here? unnatural */
```

# and in MRI rubygems: lib/rubygems/core_ext/kernel_require.rb

```rb
  def require path
    RUBYGEMS_ACTIVATION_MONITOR.enter

    path = path.to_path if path.respond_to? :to_path

    spec = Gem.find_unresolved_default_spec(path)
    if spec
      Gem.remove_unresolved_default_spec(spec)
      gem(spec.name)
    end

    # If there are no unresolved deps, then we can use just try
    # normal require handle loading a gem from the rescue below.

    if Gem::Specification.unresolved_deps.empty? then
      begin
        RUBYGEMS_ACTIVATION_MONITOR.exit
        return gem_original_require(path)
      ensure
        RUBYGEMS_ACTIVATION_MONITOR.enter
      end
    end

```

# moar rubygems requiring

```rb

    # If +path+ is for a gem that has already been loaded, don't
    # bother trying to find it in an unresolved gem, just go straight
    # to normal require.
    #--
    # TODO request access to the C implementation of this to speed up RubyGems

    spec = Gem::Specification.stubs.find { |s|
      s.activated? and s.contains_requirable_file? path
    }

    begin
      RUBYGEMS_ACTIVATION_MONITOR.exit
      return gem_original_require(path)
    ensure
      RUBYGEMS_ACTIVATION_MONITOR.enter
    end if spec

    # Attempt to find +path+ in any unresolved gems...

    found_specs = Gem::Specification.find_in_unresolved path

    # If there are no directly unresolved gems, then try and find +path+
    # in any gems that are available via the currently unresolved gems.
    # For example, given:
    #
    #   a => b => c => d
    #
    # If a and b are currently active with c being unresolved and d.rb is
    # requested, then find_in_unresolved_tree will find d.rb in d because
    # it's a dependency of c.
    #
    if found_specs.empty? then
      found_specs = Gem::Specification.find_in_unresolved_tree path

      found_specs.each do |found_spec|
        found_spec.activate
      end

    # We found +path+ directly in an unresolved gem. Now we figure out, of
    # the possible found specs, which one we should activate.
    else

      # Check that all the found specs are just different
      # versions of the same gem
      names = found_specs.map(&:name).uniq

      if names.size > 1 then
        raise Gem::LoadError, "#{path} found in multiple gems: #{names.join ', '}"
      end

```

# moar rubygems requiring

```rb
      # Ok, now find a gem that has no conflicts, starting
      # at the highest version.
      valid = found_specs.select { |s| s.conflicts.empty? }.last

      unless valid then
        le = Gem::LoadError.new "unable to find a version of '#{names.first}' to activate"
        le.name = names.first
        raise le
      end

      valid.activate
    end

    begin
      RUBYGEMS_ACTIVATION_MONITOR.exit
      return gem_original_require(path)
    ensure
      RUBYGEMS_ACTIVATION_MONITOR.enter
    end
  rescue LoadError => load_error
    if load_error.message.start_with?("Could not find") or
        (load_error.message.end_with?(path) and Gem.try_activate(path)) then
      begin
        RUBYGEMS_ACTIVATION_MONITOR.exit
        return gem_original_require(path)
      ensure
        RUBYGEMS_ACTIVATION_MONITOR.enter
      end
    end

    raise load_error
  ensure
    RUBYGEMS_ACTIVATION_MONITOR.exit
  end
```

# Common themes

-- Finding files
-- Loading code / evaluating Ruby
-- Am I in a thread? Don't load unless I'm the only one doing it

# Finding files

--- LOAD_PATH and GEM_PATH

# LOAD_PATH and GEM_PATH 

```bash
➜  jruby git:(master) ruby -e 'puts $LOAD_PATH'                                                    
/Users/Joseph.Winter/.rvm/rubies/ruby-2.0.0-p353/lib/ruby/site_ruby/2.0.0
/Users/Joseph.Winter/.rvm/rubies/ruby-2.0.0-p353/lib/ruby/site_ruby/2.0.0/x86_64-darwin13.1.0
/Users/Joseph.Winter/.rvm/rubies/ruby-2.0.0-p353/lib/ruby/site_ruby
/Users/Joseph.Winter/.rvm/rubies/ruby-2.0.0-p353/lib/ruby/vendor_ruby/2.0.0
/Users/Joseph.Winter/.rvm/rubies/ruby-2.0.0-p353/lib/ruby/vendor_ruby/2.0.0/x86_64-darwin13.1.0
/Users/Joseph.Winter/.rvm/rubies/ruby-2.0.0-p353/lib/ruby/vendor_ruby
/Users/Joseph.Winter/.rvm/rubies/ruby-2.0.0-p353/lib/ruby/2.0.0
/Users/Joseph.Winter/.rvm/rubies/ruby-2.0.0-p353/lib/ruby/2.0.0/x86_64-darwin13.1.0
➜  jruby git:(master) ruby -e 'require "rubygems"; puts ENV["GEM_PATH"]'     
/Users/Joseph.Winter/.rvm/gems/ruby-2.0.0-p353:/Users/Joseph.Winter/.rvm/gems/ruby-2.0.0-p353@global
```

# require can load non-ruby files

--- require 'ruby_source_or_compiled_shared_object_or_DLL'
--- .rb, .so, .dll
--- JRuby can't do DLLs, but can do JAR files, which are better anyway

# Loading Code / evaluating Ruby

--- load('load') # there's a lot to that
--- require is just Kernel#load w/o re-loading already loaded code

# Am I in a thread?

--- What harm could come from loading not synchronizing code loading?
--- require promises that it won't load twice, that'd fail
--- What else?


# What require doesn't do, but might be nice

- Namespacing (help avoid name collision, play nice)
-- import Net::HTTP as Aychtttp
-- import HTTP as the_http_gem

# me

@jwinter

# last slide

Bye!
