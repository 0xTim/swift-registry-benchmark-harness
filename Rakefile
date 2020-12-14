require 'rake/clean'

require 'json'
require 'uri'
require 'dotenv/tasks'

# Disable FileUtils logging statements
verbose(false)

CLEAN << '.build'
CLEAN << File.expand_path('~/Library/Caches/org.swift.swiftpm/repositories/')

CLEAN << 'Package.resolved'
desc 'Resolve package dependencies'
file 'Package.resolved' => ['Package.swift'] do
  `swift package resolve`
end

CLOBBER << 'dependencies.json'
desc 'Generate a list of dependencies'
file 'dependencies.json' => ['Package.resolved'] do |t|
  resolved = JSON(File.read(t.prerequisites.first))

  dependencies = []
  resolved['object']['pins'].each do |pin|
    dependencies << {
      url: pin['repositoryURL'],
      version: pin['state']['version']
    }
  end

  File.write(t.name, dependencies.to_json)
end

CLEAN << '.swiftpm/config'
directory '.swiftpm'

desc "Set mirror URLs for dependencies to go through package registry"
file '.swiftpm/config' => [:dotenv, '.swiftpm', 'dependencies.json'] do |t|
  raise 'Missing environment variable SWIFT_REGISTRY_URL' unless ENV['SWIFT_REGISTRY_URL']

  dependencies = JSON(File.read(t.prerequisites.last))

  dependencies.each do |dependency|
    original_url = URI(dependency['url'].sub(/\.git$/, ''))
    mirror_url = URI(ENV['SWIFT_REGISTRY_URL'])
    mirror_url.path = ('/' + original_url.host + original_url.path).squeeze('/')
    `swift package config set-mirror --original-url #{original_url} --mirror-url #{mirror_url}`
  end
end

CLOBBER << '.index'
desc "Generate a package registry index from the list of dependencies"
directory '.index' => ['dependencies.json'] do |t|
  `swift registry init --index #{t.name}`

  dependencies = JSON(File.read(t.prerequisites.first))

  dependencies.each do |dependency|
    url = URI(dependency['url'].sub(/\.git$/, ''))
    package = url.host + url.path
    version = dependency['version'].sub(/^v?/, '')

    `swift registry publish #{package} #{version} --index #{t.name}`
  end
end

CLEAN << 'spm'
desc 'Generate a shim for Swift package manager'
file 'spm' => [:dotenv] do |t|
  raise 'Missing environment variable SWIFT_PACKAGE_MANAGER_BUILD_PATH' unless ENV['SWIFT_PACKAGE_MANAGER_BUILD_PATH']

  build_path = File.expand_path(ENV['SWIFT_PACKAGE_MANAGER_BUILD_PATH'])
  subcommands = %w[build package test run]

  File.open(t.name, 'w') do |f|
    f << <<~SH
      #!/usr/bin/env bash

      set -eo pipefail

      if [ -z "$1" ]; then
          echo "missing subcommand [#{subcommands.join('|')}]"
          exit 1
      fi

      BUILD_PATH="#{build_path}"

    SH

    f.puts 'case "$1" in'
    subcommands.each do |c|
      f.puts %(#{c}\) "$BUILD_PATH/swift-#{c}" "${@:2}" ;;)
    end
    f.puts 'esac'
  end

  chmod '+x', t.name
end

namespace :registry do
  desc 'Start running the package registry server'
  task start: ['.index', 'registry:stop'] do |t|
    IO.popen(['swift-registry', 'serve', '--index', t.prerequisites.first])
    sleep 1
  end

  desc 'Stop the package registry server'
  task :stop do
    system 'pkill -f swift-registry'
  end
end

namespace :benchmark do
  desc 'Benchmark the performance of resolving with Git repositories'
  task repository: [:clean] do
    puts command = 'time swift run'
    system command
  end

  desc 'Benchmark the performance of resolving with a package registry'
  task registry: [:clean, '.swiftpm/config', 'spm'] do
    rm 'Package.resolved' if File.exist? 'Package.resolved'
    puts command = 'time ./spm run --enable-package-registry'
    system command
  end

  task all: %i[repository registry]
end

Rake::Task['benchmark:registry'].enhance ['registry:start']
Rake::Task['benchmark:registry'].enhance do
  Rake::Task['registry:stop'].invoke
end

desc 'Benchmark the performance of dependency resolution with repositories and a registry'
task benchmark: 'benchmark:all'
