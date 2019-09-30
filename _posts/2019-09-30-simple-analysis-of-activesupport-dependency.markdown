---
layout: post
title: "The back of Rails autoload"
date: 2019-09-30 08:00:00 +0800
categories: ruby-on-rails
---
## Preface
Rails can autoload constant even though you didn't require the corresponding ruby file.It relies on `ActiveSupport::Dependencies` module before Rails 6.Rails 6 use Zeitwerk instead.

First, I will review the ruby constant lookup mechanism.Second, I will explore the source code of `ActiveSupport::Dependencies`

## Ruby constant lookup
#### Module.nesting
First, ruby will check from current lexical scope to top scope recursively until it find corresponding constant.
{% highlight ruby %}
module A
  module B end
end

module A
  module C
    # ruby first check A::C::B, doesn't find
    # then check A::B, can find
    B
    puts 'can find B inside c'
  end
end

module A::D
  # ruby first check A::D::B, doesn't find
  # then top scope ::B, doesn't find either
  # then raise error
  B
rescue NameError
  puts 'can not find B inside D'
end

module A
  module B
    module C
      # use Module.nesting can show scopes from current to top
      puts Module.nesting
      # print [A::B::C, A::B, A]
    end
  end
end
{% endhighlight %}

#### Ancestors
If ruby can't find constant through Module.nesting, ruby will check current open class or module's ancestors
{% highlight ruby %}
class A
  module B; end
end

class C < A
  # ruby first check Module.nesting, can not find B
  # then, check the ancestor of C, that is A. can find B
  B
  puts 'can find B inside C'
end

class C < A
  class D
    # ruby first check Module.nesting, can't find B
    # then check ancestor of D, that is Object, can't find B
    # all ancestors of D can't find B
    # raise error
    B
  rescue NameError
    puts 'can not find B inside D'
  end
end

# use .ancestors method to show all ancestors
puts C.ancestors
# print [C, A, Object, Kernel, BasicObject]
{% endhighlight %}

## ActiveSupport::Dependencies
If ruby can't find constant, there is a callback `const_missing`, just like `method_missing`.Rails will guess the corresponding file name accorrding to constant name, and then search through every directory inside `$autoload_path` to find the corresponding file to require.
{% highlight ruby %}
# https://github.com/rails/rails/blob/2f76256127d35cfbfaadf162e4d8be2d0af4e453/activesupport/lib/active_support/dependencies.rb

module ActiveSupport
  module Dependencies
    extend self
    module ModuleConstMissing
      def const_missing(const_name)
        # overriding const_missing
        # try to find missing constant
        # for example, from_mod is A::B, const_name is C
        from_mod = anonymous? ? guess_for_anonymous(const_name) : self
        Dependencies.load_missing_constant(from_mod, const_name)
      end
    end

    def load_missing_constant(from_mod, const_name)
      # skip

      # qualified_name_for return something like A::B::C
      qualified_name = qualified_name_for from_mod, const_name
      
      # path_suffix is something like a/b/c
      path_suffix = qualified_name.underscore

      # search_for_file: try to find file_name like a/b/c.rb in $autoload_path
      file_path = search_for_file(path_suffix)

      if file_path
        expanded = File.expand_path(file_path)
        expanded.sub!(/\.rb\z/, "".freeze)
        # skip

        # if find the file, require it
        require_or_load(expanded, qualified_name)
        # skip  

      # if there is a directory instead of file named a/b/c, create a module named A::B::C manually
      elsif mod = autoload_module!(from_mod, const_name, qualified_name, path_suffix)
        return mod

      elsif (parent = from_mod.parent) && parent != from_mod &&
            ! from_mod.parents.any? { |p| p.const_defined?(const_name, false) }
        # the upper `elsif` is to check `Qualified References`
        # see https://guides.rubyonrails.org/v5.2/autoloading_and_reloading_constants.html#autoloading-algorithms-qualified-references to know more about `Qualified References`

        begin
          # if did not find, recursively search in parent scope of from_mod, that is A::C. Continually call `load_missing_constant`
          return parent.const_missing(const_name)
        rescue NameError => e
          raise unless e.missing_name? qualified_name_for(parent, const_name)
        end
      end

      # if still not find after recursively search in all parent scope
      # raise error
      name_error = NameError.new("uninitialized constant #{qualified_name}", const_name)
      name_error.set_backtrace(caller.reject { |l| l.starts_with? __FILE__ })
      raise name_error
    end

    def qualified_name_for(mod, name)
      mod_name = to_constant_name mod
      mod_name == "Object" ? name.to_s : "#{mod_name}::#{name}"
    end

    def to_constant_name(desc)
      case desc
      when String then desc.sub(/^::/, "")
      when Symbol then desc.to_s
      when Module
        desc.name ||
          raise(ArgumentError, "Anonymous modules have no name to be referenced by")
      else raise TypeError, "Not a valid constant descriptor: #{desc.inspect}"
      end
    end
    
    # check if there is a file named path_suffix in $autoload_path
    def search_for_file(path_suffix)
      path_suffix = path_suffix.sub(/(\.rb)?$/, ".rb".freeze)

      autoload_paths.each do |root|
        path = File.join(root, path_suffix)
        return path if File.file? path
      end
      nil
    end

    # check if there is a directory named path_suffix in $autoload_path
    def autoloadable_module?(path_suffix)
      autoload_paths.each do |load_path|
        return load_path if File.directory? File.join(load_path, path_suffix)
      end
      nil
    end

    # create a module named const_name manually
    def autoload_module!(into, const_name, qualified_name, path_suffix)
      return nil unless base_path = autoloadable_module?(path_suffix)
      mod = Module.new
      into.const_set const_name, mod

      # skip

      mod
    end
  end
end
{% endhighlight %}

#### Summary
As we can see from above, `ActiveSupport::Dependencies` overriding const_missing. If A::B::C is missing, Rails try to find a/b/c.rb in `$autoload_path`.If not find a/b/c.rb, Rails tries to B's parent scope recursively, that is A::C and try to find a/c.rb in `$autoload_path`. If still not find after tried all parents scope, raise error.
