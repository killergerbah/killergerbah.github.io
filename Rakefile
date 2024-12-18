require "bundler/setup"
require "jekyll/task/i18n"

Jekyll::Task::I18n.define do |task|
  # Set translate target locales.
  task.locales = ["ja"]
  # Set all *.md texts as translate target contents.
  task.files = Rake::FileList["**/*.markdown"]
  # Remove internal files from target contents.
#  task.files -= Rake::FileList["_*/**/*.md"]
  # Remove translated files from target contents.
  task.locales.each do |locale|
    task.files -= Rake::FileList["#{locale}/**/*.markdown"]
  end
end

task :default => "jekyll:i18n:translate"
