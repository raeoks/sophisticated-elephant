development_directory = 'build/development'
staging_directory     = 'build/staging'
production_directory  = 'build/production'
public_directory      = 'public'
source_directory      = 'source'
posts_directory       = '_posts'
new_post_extension    = 'markdown'

development_config = 'config/development.yml'
staging_config     = 'config/staging.yml'
production_config  = 'config/production.yml'

server_port = '9001'

def get_stdin(message)
  print message
  STDIN.gets.chomp
end

def ask(message, valid_options)
  if valid_options
    answer = get_stdin("#{message} #{valid_options.to_s.gsub(/"/, '').gsub(/, /, '/')} ") until valid_options.include?(answer)
  else
    answer = get_stdin(message)
  end
  answer
end

desc 'generate jekyll site'
task :generate do
  puts '## Generating Site with Jekyll'
  rm_rf development_directory
  mkdir_p development_directory
  %w(config.ru Gemfile Gemfile.lock puma.rb Procfile).each do |filename|
    cp filename, "#{development_directory}/#{filename}"
  end
  system "jekyll build --config #{development_config}"
end

desc 'watch the site and regenerate when it changes'
task :watch => :generate do
  puts '## Starting to watch source with Jekyll.'
  jekyll_pid = Process.spawn({'JEKYLL_ENV' => 'preview'}, "jekyll build --watch --config #{development_config}")
  trap('INT') do
    [jekyll_pid].each { |pid| Process.kill(9, pid) rescue Errno::ESRCH }
    exit 0
  end
  [jekyll_pid].each { |pid| Process.wait(pid) }
end

desc 'preview the site in a web browser'
task :preview => :generate do
  puts '## Starting to watch source with Jekyll.'
  jekyll_pid = Process.spawn({'JEKYLL_ENV' => 'preview'}, "jekyll build --watch --config #{development_config}")
  cd "#{development_directory}" do
    puts "## Starting Rack on port #{server_port}"
    rackup_pid = Process.spawn("rackup --port #{server_port}")
    trap('INT') do
      [jekyll_pid, rackup_pid].each { |pid| Process.kill(9, pid) rescue Errno::ESRCH }
      exit 0
    end
    [jekyll_pid, rackup_pid].each { |pid| Process.wait(pid) }
  end
end

desc "begin a new post in #{source_directory}/#{posts_directory}"
task :new_post, :title do |task, arguments|
  if arguments.title
    title = arguments.title
  else
    title = get_stdin('Enter a title for your post: ')
  end
  raise "### You haven't set anything up yet. First run `rake install` to set up an Octopress theme." unless File.directory?(source_directory)
  mkdir_p "#{source_directory}/#{posts_directory}"
  filename = "#{source_directory}/#{posts_directory}/#{Time.now.strftime('%Y-%m-%d')}-#{title.to_url}.#{new_post_extension}"
  if File.exist?(filename)
    abort('rake aborted!') if ask("#{filename} already exists. Do you want to overwrite?", %w(y n)) == 'n'
  end
  puts "Creating new post: #{filename}"
  open(filename, 'w') do |post|
    post.puts '---'
    post.puts 'layout: post'
    post.puts "title: \"#{title.gsub(/&/, '&amp;')}\""
    post.puts "date: #{Time.now.strftime('%Y-%m-%d %H:%M:%S %z')}"
    post.puts 'comments: true'
    post.puts 'categories: '
    post.puts '---'
  end
end

desc 'clean out caches: .pygments-cache, .gist-cache, .sass-cache'
task :clean do
  rm_rf [Dir.glob('.pygments-cache/**'), Dir.glob('.gist-cache/**'), Dir.glob('.sass-cache/**'),
         Dir.glob("#{source_directory}/.jekyll-assets-cache/**")]
end

desc 'deploy to production'
task :deploy do
  puts '## Generating Site with Jekyll'
  rm_rf "#{staging_directory}/#{public_directory}"
  mkdir_p production_directory
  %w(config.ru Gemfile Gemfile.lock puma.rb Procfile).each do |filename|
    cp filename, "#{production_directory}/#{filename}"
  end
  system "jekyll build --config #{production_config}"
  puts '## Deploying to Heroku'
  cd "#{production_directory}" do
    system 'git add .'
    system 'git add -u'
    puts "\n## Committing: Site updated at #{Time.now.utc}"
    message = "Site updated at #{Time.now.utc}"
    system "git commit -m '#{message}'"
    puts "\n## Pushing generated #{production_directory} website"
    system 'git push heroku master'
    puts "\n## Heroku deploy complete"
  end
end

desc 'deploy to staging'
task :stage do
  puts '## Generating Site with Jekyll'
  rm_rf "#{staging_directory}/#{public_directory}"
  mkdir_p staging_directory
  %w(config.ru Gemfile Gemfile.lock puma.rb Procfile).each do |filename|
    cp filename, "#{staging_directory}/#{filename}"
  end
  system "jekyll build --config #{staging_config}"
  puts '## Deploying to Heroku'
  cd "#{staging_directory}" do
    system 'git add .'
    system 'git add -u'
    puts "\n## Committing: Site updated at #{Time.now.utc}"
    message = "Site updated at #{Time.now.utc}"
    system "git commit -m '#{message}'"
    puts "\n## Pushing generated #{staging_directory} website"
    system 'git push heroku master'
    puts "\n## Heroku deploy complete"
  end
end

desc 'copy dot files for deployment'
task :copy_dot, :source, :destination do |task, arguments|
  FileList["#{arguments.source}/**/.*"].exclude('**/.', '**/..', '**/.DS_Store', '**/._*').each do |file|
    cp_r file, file.gsub(/#{arguments.source}/, "#{arguments.destination}") unless File.directory?(file)
  end
end

desc 'setup development, staging and production builds deployments'
task :setup, :appname do |task, arguments|
  if arguments.appname
    appname = arguments.appname
  else
    puts 'Enter a Heroku Application Name'
    puts 'e.g. sophisticated-elephant, brave-mice ...'
    appname = get_stdin('Heroku Application Name: ')
  end

  system 'bundle install'

  [staging_directory, production_directory].each do |directory|
    mkdir_p "#{directory}/#{public_directory}"
    cp "#{source_directory}/404.html", "#{directory}/#{public_directory}"
    %w(config.ru Gemfile Gemfile.lock puma.rb Procfile).each do |filename|
      cp filename, "#{directory}/#{filename}"
    end
  end

  [staging_directory, production_directory].zip(["#{appname}-staging", appname]).each do |directory, heroku_appname|
    cd "#{directory}" do
      system 'git init'
      system "heroku apps:create #{heroku_appname}"
      system 'git add .'
      system "git commit -m \"initial commit\""
      system 'git config branch.master.remote heroku'
      system 'git push --set-upstream heroku master'
    end
  end

  puts "\n---\n## Now you can deploy to http://#{appname}-staging.herokuapp.com with `rake stage` ##"
  puts "\n---\n## Now you can deploy to http://#{appname}.herokuapp.com with `rake deploy` ##"
end

desc 'list tasks'
task :list do
  puts "Tasks: #{(Rake::Task.tasks - [Rake::Task[:list]]).join(', ')}"
  puts "(type rake -T for more detail)\n\n"
end
