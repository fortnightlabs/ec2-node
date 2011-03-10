load 'deploy' if respond_to?(:namespace) # cap2 differentiator
Dir['vendor/plugins/*/recipes/*.rb'].each { |plugin| load(plugin) }

default_run_options[:pty] = true

set :application, "#{ENV['NODE_PROCESS']}"
set :repository,  "git://github.com/fortnightlabs/lazeroids.git"
set :scm, :git

set :user, "app"

role :app, "#{application}"

namespace :deploy do
  task :start, :roles => :app do
    run "#{try_sudo} start #{application}"
  end

  task :stop, :roles => :app do
    run "#{try_sudo} stop #{application}"
  end

  task :restart, :roles => :app, :except => { :no_release => true } do
    run "#{try_sudo} restart #{application}"
  end
end

namespace :npm do
  task :install, :roles => :app do
    run "cd #{release_path} && /u/apps/.nodelocal/bin/npm install ."
  end
end
after "deploy:update_code", "npm:install"

namespace :with_sudo do
  task :on do
    set :use_sudo, true
  end
  task :off do
    set :use_sudo, false
  end
end
before "deploy:setup", "with_sudo:off"
after "deploy:setup", "with_sudo:on"
