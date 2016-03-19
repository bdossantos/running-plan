require 'html-proofer'
require 'image_optim'
require 'English'

LIBS_DIR = '_libs'
BUILD_DIR = '_build'
HTML_COMPRESSOR = `which htmlcompressor`.chomp
YUI_COMPRESSOR = `which yuicompressor`.chomp
BOWER = `which bower`.chomp
WGET = `which wget`.chomp
BUCKET = 's3://bds.run'
RASPBERRY = 'pi@192.168.1.2:/srv/http/bds.run'
JEKYLL_ENV = ENV['JEKYLL_ENV'] || 'development'

desc 'Jekyll build'
task :jekyll_build do
  puts '--> Jekyll build'
  system "rm -rf #{BUILD_DIR}"
  config = File.exist?("_config_#{JEKYLL_ENV}.yml") ? ",_config_#{JEKYLL_ENV}.yml" : nil
  system "jekyll build -d #{BUILD_DIR} --config _config.yml#{config}"
end

desc 'Begin a new post'
task :post do
  title = ENV['title'] || 'new-post'
  tags = ENV['tags'] || '[]'
  category = ENV['category'] || ''
  category = "\"#{category.gsub(/-/, ' ')}\"" unless category.empty?
  slug = title.downcase.strip.gsub(' ', '-').gsub(/[^\w-]/, '')

  begin
    date = (ENV['date'] ? Time.parse(ENV['date']) : Time.now)\
           .strftime('%Y-%m-%d')
  rescue
    puts 'Error - date format must be YYYY-MM-DD.'
    exit 1
  end

  filename = File.join('_posts', "#{date}-#{slug}.md")
  if File.exist?(filename)
    abort('rake aborted!') if ask("#{filename} already exists. " +
                              'Do you want to overwrite?', %w(y n)) == 'n'
  end

  puts "Creating new post: #{filename}"
  open(filename, 'w') do |post|
    post.puts '---'
    post.puts 'layout: post'
    post.puts "title: \"#{title.gsub(/-/, ' ')}\""
    post.puts 'description: ""'
    post.puts "category: #{category}"
    post.puts "tags: #{tags}"
    post.puts '---'
  end
end

desc 'Notify Google of the new sitemap'
task :sitemap do
  begin
    require 'net/http'
    require 'uri'
    puts '* Pinging Google about our sitemap'
    Net::HTTP.get('www.google.com', '/webmasters/tools/ping?sitemap=' +
               URI.escape('http://bds.run/sitemap.xml'))
  rescue LoadError
    puts '! Could not ping Google about our sitemap, because Net::HTTP ' \
         'or URI could not be found.'
  end
end

desc 'Bower install'
task :bower_install do
  puts '--> Grab front-end packages with Bower'
  system "#{BOWER} install"
end

desc 'Minify all html'
task :minify_html do
  puts '--> Minifying html'
  system "find #{BUILD_DIR} -type f -name '*.html' " +
    "| xargs -I '%' -P 4 -n 1 #{HTML_COMPRESSOR} --remove-intertag-spaces " +
    "--compress-css --compress-js --js-compressor yui '%' -o '%'"
end

desc 'Gzip'
task :gzip, [:ext] => [:gzip_all] do |t, args|
  puts "--> GZipping '#{args.ext}'"
  system "find #{BUILD_DIR} -type f -name '*.#{args.ext}' -print0 | " +
         "xargs -0 -I % -P 4 -n 1 sh -c 'gzip -9 < % > %.gz'"
end

desc 'GZip All'
task :gzip_all do
  Rake::Task[:gzip].execute('html')
  Rake::Task[:gzip].execute('css')
  Rake::Task[:gzip].execute('js')
end

desc 'Image optimization'
task :image_optimization do
  puts '--> Optimize images'
  image_optim = ImageOptim.new(:pngout => false, :svgo => false)
  image_optim.optimize_images!(Dir['**/*.png', '**/*.jpg']) do |u, o|
    puts "#{u} => #{o}" if o
  end
end

desc 'Test for 404s'
task :check_html do
  puts '--> Check for broken links'
  HTMLProofer.check_directory(
    BUILD_DIR,
    {
      :ext => '.html',
      :parallel => { :in_processes => 4 },
      :url_ignore => [ '#', '/twitter.com/', '/disqus.com/' ],
      :validate_html => false,
      :disable_external => true
    }
  ).run
end

desc 'upload to s3'
task :upload_to_s3 do
  puts '--> Sync media files first + set cache expires'
  system "s3cmd sync \
      --no-preserve \
      --guess-mime-type \
      --acl-public \
      --exclude '*.*' \
      --include '*.png' \
      --include '*.jpg' \
      --include '*.ico' \
      --add-header='Expires: Sat, 20 Nov 2020 18:46:39 GMT' \
      --add-header='Cache-Control: max-age=6048000' \
      #{BUILD_DIR}/ \
      #{BUCKET}"

  puts '--> Sync Javascript and CSS assets next (Cache: expire in 1 week)'
  system "s3cmd sync \
      --no-preserve \
      --guess-mime-type \
      --acl-public \
      --exclude '*.*' \
      --include '*.css' \
      --include '*.js' \
      --add-header='Cache-Control: max-age=604800' \
      --add-header='Vary: Accept-Encoding' \
      #{BUILD_DIR}/ \
      #{BUCKET}"

  puts '--> Sync Gzipped files'
  system "s3cmd sync \
      --no-preserve \
      --guess-mime-type \
      --acl-public \
      --exclude '*.*' \
      --include '*.gz' \
      --add-header='Cache-Control: max-age=604800' \
      --add-header='Content-Encoding: gzip' \
      #{BUILD_DIR}/ \
      #{BUCKET}"

  puts '--> Sync everything else, but ignore the assets!'
  system "s3cmd sync \
      --no-preserve \
      --guess-mime-type \
      --acl-public \
      --exclude '*.*' \
      --exclude '.htaccess' \
      --include '*.html' \
      --add-header='Cache-Control: max-age=7200, must-revalidate' \
      --add-header='Vary: Accept-Encoding' \
      #{BUILD_DIR}/ \
      #{BUCKET}"

  puts '--> Sync remaining files & delete removed'
  system "s3cmd sync \
      --no-preserve \
      --acl-public \
      --delete-removed \
      #{BUILD_DIR}/ \
      #{BUCKET}"
end

desc 'Fix files permissions'
task :fix_files_permissions do
  puts '--> Fix files permissions'
  system "find #{BUILD_DIR} -type f | xargs chmod 644"
  system "find #{BUILD_DIR} -type d | xargs chmod 755"
end

desc 'Upload to Raspberry Pi'
task :upload_to_pi do
  puts '--> Upload to Raspberry Pi'
  system "rsync -zav --delete #{BUILD_DIR}/ #{RASPBERRY}/"
end

desc 'Publish to Github Pages'
task :publish_to_gh_pages do
  puts '--> Publish to Github Pages'

  fail 'Run deploy task' unless File.exist?(BUILD_DIR)

  sha = `git rev-parse HEAD`.strip
  Dir.chdir(BUILD_DIR) do
    system 'rm -rf .git && git init && git add . >/dev/null 2>&1'

    if ENV['EMAIL']
      system "git config --global user.email '#{ENV['EMAIL']}'"
      system 'git config --global user.name "Travis-CI"'
    end

    fail 'Failed to commit' \
      unless system "git commit --allow-empty -m 'Updating to #{sha} [skip-ci]' >/dev/null 2>&1"

    if ENV['TOKEN']
      # hide output
      output = system "git push -f https://#{ENV['TOKEN']}:x-oauth-basic@github.com/bdossantos/runner.sh master:gh-pages >/dev/null 2>&1"
      output = nil
    else
      system 'git push -f https://github.com/bdossantos/runner.sh master:gh-pages >/dev/null 2>&1'
    end
  end
end

desc 'Full deployement task'
task :deploy do
  puts '--> Start Deploy'
  Rake::Task['bower_install'].invoke
  Rake::Task['jekyll_build'].invoke
  Rake::Task['minify_html'].invoke
  Rake::Task['gzip_all'].invoke
  Rake::Task['image_optimization'].invoke
  Rake::Task['fix_files_permissions'].invoke
  Rake::Task['publish_to_gh_pages'].invoke
  Rake::Task['check_html'].invoke
  puts '--> End'
end

desc 'Clean activities CSV'
task :clean_activites_csv do
  ACTIVITIES = './_data/activities/'

  # Clean whitespaces
  system "find #{ACTIVITIES} -type f -name '*.csv' | \
          xargs -n 1 -P 4 sed -i -e 's/^[ \t]*//' -e 's/[ \t]*$//'"

  # Clean last blank line
  system "find #{ACTIVITIES} -type f -name '*.csv' | \
          xargs -n 1 -P 4 sed -i '/^ *$/d'"

  # Clean useless line
  system "find #{ACTIVITIES} -type f -name '*.csv' | \
          xargs -n 1 -P 4 sed -i '/Activities by bdossantos/d'"
end
