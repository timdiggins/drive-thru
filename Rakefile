if File.exists? File.join(File.dirname(__FILE__), 'config', 'rake.rb') 
  require File.join(File.dirname(__FILE__), 'config', 'rake')
else 
  HOST = "undefined"
  puts "You need to do 'rake init_drive_thru' before anything else will work\n"
end
load 'sprinkle_chef/Rakefile'
load 'lib/dns.rb'
load 'lib/chef-repo.rb'

desc "Create various default versions from templates"
task :init_drive_thru do
  config_src = File.join(File.dirname(__FILE__), "drive-thru-templates", 'config')
  config_dest = File.join(File.dirname(__FILE__), 'config')
  FileUtils.cp_r config_src, config_dest
  site_modules_src = File.join(config_dest, 'site-cookbooks')
  site_modules_dest = File.join(File.dirname(__FILE__), 'site-cookbooks')
  FileUtils.ln_s site_modules_src, site_modules_dest
  puts "initialized templates - now fill in the right data and do git init within config\n"
end

desc "Create dna.json from dna.rb file"
task :create_dna  do
  dna_file = File.join(File.dirname(__FILE__), "config", "dna.rb")
  raise "There is no config/dna.rb file! Take care of that!" unless File.exists? dna_file 
  sh "ruby #{dna_file}"
end

task :update_config => [:update_drive_thru_info,:create_dna] do
  remote("mkdir -p /etc/chef")
  sh "#{RSYNC} #{TOPDIR}/config/* #{HOST_LOGIN}:/etc/chef/"
  File.delete(File.dirname(__FILE__) + "/config/dna.json")
end

def git_version
    'git log -1 --format=oneline'
end

task :update_drive_thru_info do
  drive_thru_version = File.join(File.dirname(__FILE__), "config", "drive_thru_version.txt")
  File.delete(drive_thru_version) if File.exists? drive_thru_version
  [
    "echo '# #{git_version}:\n'",
    "#{git_version}",
    "echo '\n\n# git submodules (.gitmodules):\n'",
    "cat #{File.join(File.dirname(__FILE__), ".gitmodules")}",
    "echo '\n\n#git submodule status:\n'",
    "git submodule status"
    ].each do |cmd|
      sh "#{cmd} >> #{drive_thru_version}"
  end
end

desc "rsync the cookbooks to #{HOST}"
task :update_cookbooks do
  remote("mkdir -p /var/chef")
  ['site-cookbooks', 'cookbooks'].each do |name|
    sh "#{RSYNC} --exclude=openldap --exclude=quick_start #{File.join(TOPDIR, name)}/ #{HOST_LOGIN}:/var/chef/#{name}"
  end
end

desc "Run chef-solo on #{HOST}"
task :run_chef_solo => [:update_config, :update_cookbooks] do
  command = "chef-solo -j /etc/chef/dna.json -c /etc/chef/solo.rb"
  command << " -l debug" if ENV['DEBUG'] == 'true' 
  remote(command)
end

def remote(cmd)
  sh "ssh #{HOST_LOGIN} '#{cmd}'"
end

task :default => :run_chef_solo

desc "Automatically initialze #{HOST} from scratch. You should only have to enter the root password once."
dependencies = %w[add_ssh_key]
dependencies << 'setup_dns' unless API_PASSWORD == ""
dependencies << 'chef:solo'
dependencies << 'run_chef_solo'
task(:initialize_host => dependencies) {}
