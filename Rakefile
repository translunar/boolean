require 'rubygems'
require 'bundler'
require 'rake'
require 'rspec/core/rake_task'
require 'rspec/core'
require 'rspec/core/rake_task'

begin
  Bundler.setup(:default, :development)
rescue Bundler::BundlerError => e
  $stderr.puts e.message
  $stderr.puts "Run `bundle install` to install missing gems"
  exit e.status_code
end

gemspec = eval(IO.read("boolean.gemspec"))

require 'rake'
require "rake/extensiontask"
Rake::ExtensionTask.new do |ext|
    ext.name = 'hypergeometric'
    ext.ext_dir = 'ext/hypergeometric'
    ext.lib_dir = 'lib/'
    ext.source_pattern = "**/*.{c,cpp, h}"
end


BASEDIR = Pathname( __FILE__ ).dirname.relative_path_from( Pathname.pwd )
SPECDIR = BASEDIR + 'spec'

VALGRIND_OPTIONS = [
        "--tool=memcheck",
        "--leak-check=full",
        "--num-callers=15",
        "--error-limit=no",
        #"--partial-loads-ok=yes",
        #"--undef-value-errors=no",
        "--track-origins=yes"#,
        #"--track-fds=yes"
]
VALGRIND_MEMORYFILL_OPTIONS = [
        "--freelist-vol=100000000",
        "--malloc-fill=6D",
        "--free-fill=66 ",
]

GDB_OPTIONS = []


RSpec::Core::RakeTask.new(:spec)

task :console do |task|
  cmd = [ 'irb', "-r './lib/boolean.rb'" ]
  run *cmd
end

task :pry do |task|
  cmd = [ 'pry', "-r './lib/boolean.rb'" ]
  run *cmd
end

task :environment do |task|
  require("./lib/boolean.rb")
end

def call_permutation_test to, from, real, i, num_each, num_this, with
  puts "Running #{i}"
  Boolean.parallel_permutation_test(to, from, real, {
    :start => i*num_each, 
    :end => (i*num_each)+num_this,
    :with => with
  })
end

task :permutation_test => :environment do |task|
  opts = YAML.load(File.read("config.yaml"))
  j = opts.delete(:j)
  n = opts.delete(:n)
  n_each = n / j

  reader   = Boolean.reader([opts[:to][1], opts[:from][1]])
  to_gpm   = Boolean.gp_matrix(*opts[:to])
  to       = to_gpm.opmatrix(reader)
  from_gpm = Boolean.gp_matrix(*opts[:from])
  from_opm = from_gpm.opmatrix(reader)
  from     = opts[:op].nil? ? from_opm : Boolean::BOPMatrix.new(from_opm, opts[:op])

  real = if File.exists?("real") || File.exists?("real.gz")
    Boolean.say_with_time "Reading existing distance matrix for real" do
      Boolean::DMatrix.read("real")
    end
  else
    Boolean.say_with_time "Generating distance matrix for real" do
      Boolean::DMatrix.new(to, from).tap { |r| r.write("real", false) }
    end
  end

  task_list = [] 
 
  j.times do |jj|
    task = Thread.new do
      n_this = jj == j-1 ? (n - (n/j).to_i*(j-1)) : (n/j).to_i
      call_permutation_test(to, from, real, jj, n_each, n_this, opts[:with])
    end
    task_list << task
  end

  task_list.each { |task| task.join }
end

#namespace :console do
#  CONSOLE_CMD = ['irb', "-r './lib/boolean.rb'"]
#  desc "Run console under GDB."
#  task :gdb => [ :compile ] do |task|
#          cmd = [ 'gdb' ] + GDB_OPTIONS
#          cmd += [ '--args' ]
#          cmd += CONSOLE_CMD
#          run( *cmd )
#  end
#
#  desc "Run console under Valgrind."
#  task :valgrind => [ :compile ] do |task|
#          cmd = [ 'valgrind' ] + VALGRIND_OPTIONS
#          cmd += CONSOLE_CMD
#          run( *cmd )
#  end
#end

task :default => :spec

def run *cmd
  sh(cmd.join(" "))
end


namespace :spec do
  # partial-loads-ok and undef-value-errors necessary to ignore
  # spurious (and eminently ignorable) warnings from the ruby
  # interpreter

  RSPEC_CMD = [ 'ruby', '-S', 'rspec', '-Ilib:ext', SPECDIR ]

  #desc "Run the spec for generator.rb"
  #task :generator do |task|
  #  run 'rspec spec/generator_spec.rb'
  #end

  desc "Run specs under GDB."
  task :gdb do |task|
    cmd = [ 'gdb' ] + GDB_OPTIONS
    cmd += [ '--args' ]
    cmd += RSPEC_CMD
    run( *cmd )
  end

  desc "Run specs under cgdb."
  task :cgdb do |task|
    cmd = [ 'cgdb' ] + GDB_OPTIONS
    cmd += [ '--args' ]
    cmd += RSPEC_CMD
    run( *cmd )
  end

  desc "Run specs under Valgrind."
  task :valgrind do |task|
    cmd = [ 'valgrind' ] + VALGRIND_OPTIONS
    cmd += RSPEC_CMD
    run( *cmd )
  end
end

