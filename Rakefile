require 'tempfile'

$PROJECT_ROOT = Dir.pwd

task :s3_config do
  File.open("#{$PROJECT_ROOT}/aws.cfg", 'w') do |f|
    f.puts "[default]"
    f.puts "aws_access_key_id = #{ENV['S3_ACCESS_KEY']}"
    f.puts "aws_secret_access_key = #{ENV['S3_SECRET_ACCESS_KEY']}"
    f.puts "region = us-east-1"
  end
  ENV['AWS_CONFIG_FILE'] = "#{$PROJECT_ROOT}/aws.cfg"
end

deps = %w(
  git-build chromedriver-build phantomjs-build python-build ruby-build ssh-build
  php-build papertrail-build java-build git-crypt-build autoconf-archive-build
  couchdb-build qt-build
)

namespace :yum do
  task :init_repo do
    $REPO_DIR = if ENV['GO_SERVER_URL'] || ENV['CI']
      File.expand_path('~/.snap-ci-yum-repo')
    else
      rm_rf   File.expand_path('../repo', __FILE__)
      File.expand_path('../repo', __FILE__)
    end
    mkdir_p $REPO_DIR
  end

  task :fetch_pkgs do
    require 'json'
    urls = []
    deps.each do |dep|
      sh("curl --silent --location --netrc https://api.snap-ci.com/project/snap-ci/#{dep}/branch/master/pipelines/latest > latest-#{dep}.json")
      data = JSON.parse(File.read("latest-#{dep}.json"))
      data['stages'].each do |stage|
        stage['workers'].each do |worker|
          worker['artifacts'].each do |artifact|
            if artifact['name'] =~ /.*\.rpm$/
              urls << artifact['_links']['self']['href']
            end
          end
        end
      end
    end

    mkdir_p 'pkg'

    cd "pkg" do
      urls.each do |url|
        sh("curl --silent --location --netrc -O #{url}")
      end
    end
  end

  task :fetchrepo do
    cd $REPO_DIR do
      sh("aws s3 mb s3://#{ENV['S3_YUM_BUCKET']}")
      sh("aws s3 sync --acl public-read --delete s3://#{ENV['S3_YUM_BUCKET']} .")
    end
  end

  task :prunerepo do
    cd $REPO_DIR do
      sh("repomanage --keep 3 --old rpm | xargs rm -fv")
    end
  end

  task :createrepo do
    sh('sudo su - -c "yum install repoview -y"')
    cd $REPO_DIR do
      mkdir_p "rpm"
      sh("cp #{$PROJECT_ROOT}/pkg/*.rpm ./rpm/")
      sh("createrepo --database --update . || createrepo --database .")
      sh("repoview --title 'Extra packages for Snap CI' .")
    end
  end

  task :uploadrepo do
    cd $REPO_DIR do
      sh("aws s3 sync --acl public-read --storage-class REDUCED_REDUNDANCY --delete . s3://#{ENV['S3_YUM_BUCKET']}")
    end
  end

  task :all => [:s3_config, :init_repo, :fetch_pkgs, :fetchrepo, :prunerepo, :createrepo, :uploadrepo]
end

namespace :yum_mirrors do
  task :init_mirrors_dir do
    $REPO_DIR = if ENV['GO_SERVER_URL'] || ENV['CI']
      File.expand_path('~/.snap-ci-yum-mirrors')
    else
      rm_rf   File.expand_path('../repo', __FILE__)
      File.expand_path('../repo', __FILE__)
    end
    mkdir_p $REPO_DIR
  end

  task :create_snapshot_folder do
    $SNAPSHOT_DIR = "#{$REPO_DIR}/#{version}"
    mkdir_p $SNAPSHOT_DIR
  end

  task :reposync do
    repo_name_list.each do |repo_name|
      tmp_repo_config = generate_repo_config_from(repo_name)
      sh "reposync -p #{$SNAPSHOT_DIR} -r #{repo_name} -c #{tmp_repo_config.path}"
    end
  end

  task :createrepo do
    sh('sudo su - -c "yum install repoview -y"')
    cd $SNAPSHOT_DIR do
      sh("createrepo --database --update .")
      sh("repoview --title 'Mirror repositories' .")
    end
  end

  task :uploadrepo do
    cd $SNAPSHOT_DIR do
      sh("aws s3 sync --acl public-read --storage-class REDUCED_REDUNDANCY . s3://#{ENV['S3_YUM_MIRRORS_BUCKET']}/#{version}")
    end
  end

  def version
    ENV['GO_PIPELINE_COUNTER'] || Time.new.to_i
  end

  def repo_name_list
    return [ENV['REPO_NAME']] if ENV['REPO_NAME']
    [
      'CentOS-Base',
      'CentOS-Debuginfo',
      'CentOS-Extras',
      'CentOS-fasttrack',
      'CentOS-Media',
      'CentOS-Updates',
      'CentOS-Vault',
      'epel',
      'mongodb',
      'mysql-community',
      'mysql-connectors-community',
      'pgdg92',
      'pgdg93',
      'pgdg94',
      'remi',
      'snap-ci-external-pkgs',
      'snap-ci'
    ]
  end

  def generate_repo_config_from(original_repo_name)
    original_repo_file = "/etc/yum.repos.d/#{original_repo_name}.repo"
    tmp_file = Tempfile.new('tmp_repo_config')
    FileUtils.cp original_repo_file, tmp_file.path
    sh "cat includepkgs >> #{tmp_file.path}"
    tmp_file
  end

  task :check_presence_of_repos do
    missing_repos = repo_name_list.reject { |repo_name| File.exists? "/etc/yum.repos.d/#{repo_name}.repo" }
    if missing_repos.any?
      abort "Could not find following repos on the current enviroment:\n#{missing_repos.join("\n")}\nIf you are running on a CI Agent, make sure it was provisioned in order to add the repo."
    end
  end

  task :all => [:check_presence_of_repos, :s3_config, :init_mirrors_dir, :create_snapshot_folder, :reposync, :createrepo, :uploadrepo]
end

namespace :apt do
  task :init_repo do
    $REPO_DIR = if ENV['GO_SERVER_URL'] || ENV['CI']
      File.expand_path('~/.snap-ci-apt-repo')
    else
      rm_rf   File.expand_path('../repo', __FILE__)
      File.expand_path('../repo', __FILE__)
    end
    mkdir_p $REPO_DIR
  end

  task :fetchrepo do
    cd $REPO_DIR do
      sh("aws s3 mb s3://#{ENV['S3_APT_BUCKET']}")
      sh("aws s3 sync --acl public-read --delete s3://#{ENV['S3_APT_BUCKET']} .")
    end
  end

  task :createrepo do
    cd $REPO_DIR do
      mkdir_p "deb"
      sh("cp #{$PROJECT_ROOT}/pkg/*.deb ./deb/")
      mkdir_p "binary"
      sh("dpkg-scanpackages deb /dev/null > binary/Packages")
      sh("gzip  -9c binary/Packages > binary/Packages.gz")
      sh("bzip2 -9c binary/Packages > binary/Packages.bz2")
    end
  end

  task :uploadrepo do
    cd $REPO_DIR do
      sh("aws s3 sync --acl public-read --storage-class REDUCED_REDUNDANCY --delete . s3://#{ENV['S3_APT_BUCKET']}")
    end
  end

  task :all => [:s3_config, :init_repo, :fetchrepo, :createrepo, :uploadrepo]
end

if File.exist?('/etc/centos-release')
  task :default => 'yum:all'
elsif File.exist?('/etc/debian_version')
  task :default => 'apt:all'
else
  $stderr.puts "Don't know what distro I'm running on -- not sure if I can build!"
end

desc "the default task"
task :default

desc 'update mirrors'
task :yum_mirrors => 'yum_mirrors:all'
