#!/usr/bin/env rake
require 'bundler/gem_tasks'

require 'rake'
require 'rspec/core/rake_task'

desc 'Run all examples'
RSpec::Core::RakeTask.new(:spec) do |t|
  t.rspec_opts = %w(--color --warnings)
end

task default: [:spec]

task :update_yaml_structure do
  require 'yaml'

  require 'pry'

  d = Dir['lib/countries/data/subdivisions/*.yaml']
  d.each do |file|
    puts "checking : #{file}"
    data = YAML.load_file(file)

    data = data.each_with_object({}) do |(k, subd), a|
      a[k] ||= {}
      a[k]['unofficial_names'] = subd.delete('names')
      a[k]['translations'] = { 'en' => subd['name'] }
      a[k]['geo'] = {
        'latitude' => subd.delete('latitude'),
        'longitude' => subd.delete('longitude'),
        'min_latitude' => subd.delete('min_latitude'),
        'min_longitude' => subd.delete('min_longitude'),
        'max_latitude' => subd.delete('max_latitude'),
        'max_longitude' => subd.delete('max_longitude')
      }

      a[k] = a[k].merge(subd)
    end
    File.open(file, 'w') { |f| f.write data.to_yaml }
    begin
  rescue
    puts "failed to read #{file}: #{$ERROR_INFO}"
  end
  end
end

desc 'Update Cache'
task :update_cache do
  require 'yaml'
  require 'i18n_data'

  codes = Dir['lib/countries/data/countries/*.yaml'].map { |x| File.basename(x, File.extname(x)) }.uniq
  data = {}

  corrections = YAML.load_file(File.join(File.dirname(__FILE__), 'lib', 'countries', 'data', 'translation_corrections.yaml')) || {}

  I18nData.languages.keys.each do |locale|
    locale = locale.downcase
    begin
      local_names = I18nData.countries(locale)
    rescue I18nData::NoTranslationAvailable
      next
    end

    # Apply any known corrections to i18n_data
    unless corrections[locale].nil?
      corrections[locale].each do |alpha2, localized_name|
        local_names[alpha2] = localized_name
      end
    end

    File.open(File.join(File.dirname(__FILE__), 'lib', 'countries', 'cache', 'locales', "#{locale}.json"), 'wb') { |f| f.write(local_names.to_json) }
  end

  codes.each do |alpha2|
    data[alpha2] ||= YAML.load_file(File.join(File.dirname(__FILE__), 'lib', 'countries', 'data', 'countries', "#{alpha2}.yaml"))[alpha2]
  end

  File.open(File.join(File.dirname(__FILE__), 'lib', 'countries', 'cache', 'countries.json'), 'wb') { |f| f.write(data.to_json) }
end

require 'geocoder'
require 'retryable'
# raise on geocoding errors such as query limit exceeded
Geocoder.configure(always_raise: :all)
# Try to geocode a given query, on exceptions it retries up to 3 times then gives up.
# @param [String] query string to geocode
# @return [Hash] first valid result or nil
def geocode(query)
  Retryable.retryable(tries: 3, sleep: ->(n) { 2**n }) do
    Geocoder.search(query).first
  end
rescue => e
  warn "Attempts exceeded for query #{query}, last error was #{e.message}"
  nil
end

desc 'Retrieve and store subdivisions coordinates'
task :fetch_subdivisions do
  require 'countries'
  # Iterate all countries with subdivisions
  ISO3166::Country.all.select(&:subdivisions?).each do |c|
    # Iterate subdivisions
    state_data = c.subdivisions.dup
    state_data.reject { |_, data| data['latitude'] }.each do |code, data|
      location = "#{data['name']}, #{c.name}"

      # Handle special geocoding cases where Google defaults to well known
      # cities, instead of the states.
      if c.alpha2 == 'US' && %w(NY WA OK).include?(code)
        location = "#{data['name']} State, United States"
      end

      next unless (result = geocode(location))
      geometry = result.geometry
      if geometry['location']
        state_data[code]['geo']['latitude'] = geometry['location']['lat']
        state_data[code]['geo']['longitude'] = geometry['location']['lng']
      end
      next unless geometry['bounds']
      state_data[code]['geo']['min_latitude'] = geometry['bounds']['southwest']['lat']
      state_data[code]['geo']['min_longitude'] = geometry['bounds']['southwest']['lng']
      state_data[code]['geo']['max_latitude'] = geometry['bounds']['northeast']['lat']
      state_data[code]['geo']['max_longitude'] = geometry['bounds']['northeast']['lng']
    end
    # Write updated YAML for current country
    File.open(File.join(File.dirname(__FILE__), 'lib', 'countries', 'data', 'subdivisions', "#{c.alpha2}.yaml"), 'w+') { |f| f.write state_data.to_yaml }
  end
end
