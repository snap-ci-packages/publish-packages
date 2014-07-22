$PROJECT_ROOT = Dir.pwd

task :s3_config do
  File.open("#{$PROJECT_ROOT}/aws.cfg", 'w') do |f|
    f.puts "[default]"
    f.puts "aws_access_key_id = #{ENV['S3_ACCESS_KEY']}"
    f.puts "aws_secret_access_key = #{ENV['S3_SECRET_KEY']}"
    f.puts "region = us-east-1"
  end
  ENV['AWS_CONFIG_FILE'] = "#{$PROJECT_ROOT}/aws.cfg"
end

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

  task :fetchrepo do
    cd $REPO_DIR do
      sh("aws s3 mb s3://#{ENV['S3_YUM_BUCKET']}")
      sh("aws s3 sync --acl public-read --delete s3://#{ENV['S3_YUM_BUCKET']} .")
    end
  end

  task :prunerepo do
    cd $REPO_DIR do
      sh("repomanage --keep 5 --old rpm | xargs rm -fv")
    end
  end

  task :createrepo do
    cd $REPO_DIR do
      mkdir_p "rpm"
      sh("cp #{$PROJECT_ROOT}/pkg/*.rpm ./rpm/")
      sh("createrepo --database --update . || createrepo --database .")
    end
  end

  task :uploadrepo do
    cd $REPO_DIR do
      sh("aws s3 sync --acl public-read --storage-class REDUCED_REDUNDANCY --delete . s3://#{ENV['S3_YUM_BUCKET']}")
    end
  end

  task :all => [:s3_config, :init_repo, :fetchrepo, :prunerepo, :createrepo, :uploadrepo]
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
