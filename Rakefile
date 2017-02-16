require 'nokogiri'
require 'open-uri'
require 'rspec/core/rake_task'
require 'yaml'
require 'zlib'

desc 'Run the specs by default'
task default: :spec

RSpec::Core::RakeTask.new(:spec)

desc "This task is used to examine .yml files for the absence of CVSS scores. If absences are detected the CVSS scores will be scraped from the relevant NVD XML and put into the yml file."

cve_dir = './tmp/checkout/chromium-vulnerabilities/cves'
nvd_base_url = 'http://nvd.nist.gov/view/vuln/detail?vulnId='
xml_base_url = 'https://static.nvd.nist.gov/feeds/xml/cve/nvdcve-2.0-'
xml_file_base = 'nvd-cve-2.0-'
xml_dir = './tmp/xml/'
xpath_cve_ref = '//vuln:cve-id[text()=\''
xpath_base_ref = '\']/../vuln:cvss/cvss:base_metrics'

task :pull_cvss do
  if Dir.exist?(cve_dir)
    #OK to continue
  else
    abort("[ERROR] Chromium CVEs not found in /tmp/checkout/chromium-vulnerabilities as expected. Please run the clone_git task if you have not already.")
  end

  loaded_cves = []
  for name in Dir.glob("#{cve_dir}/*.yml")
    loaded_cve = begin
      YAML.load(File.open(name))
    rescue ArgumentError => e
      puts "Could not parse YAML: #{e.message}"
    end
    loaded_cves.push(loaded_cve)
  end

  for cve in loaded_cves
    if cve['cvss'].nil?
      year = cve['CVE'].match(/CVE-([0-9]{4})-[0-9]{4}/)[1]

      #Create XML Directory
      unless Dir.exist?(xml_dir)
        Dir.mkdir(xml_dir)
      end

      #Download and unpack XMLs that do not exist
      unless File.exist?(xml_dir + xml_file_base + year + '.xml')
        download = open(xml_base_url + year + '.xml.gz')
        IO.copy_stream(download, xml_dir + xml_file_base + year + '.xml.gz')
        Zlib::GzipReader.open(xml_dir + xml_file_base + year + '.xml.gz') do | input_stream |
          File.open(xml_dir + xml_file_base + year + '.xml', "w") do | output_stream |
            IO.copy_stream(input_stream, output_stream)
          end
        end
        File.delete(xml_dir + xml_file_base + year + '.xml.gz')
      end

      #Pull information from XML
      xml_document = Nokogiri::XML(File.open(xml_dir + xml_file_base + year + '.xml'))
      xpath_base_query = xpath_cve_ref + cve['CVE'] + xpath_base_ref
      cvss_score = xml_document.xpath(xpath_base_query + '/cvss:score')
      access_vector = xml_document.xpath(xpath_base_query + '/cvss:access-vector')
      access_complexity = xml_document.xpath(xpath_base_query + '/cvss:access-complexity')
      authentication = xml_document.xpath(xpath_base_query +'/cvss:authentication')
      confidentiality_impact = xml_document.xpath(xpath_base_query +'/cvss:confidentiality-impact')
      integrity_impact = xml_document.xpath(xpath_base_query + '/cvss:integrity-impact')
      availability_impact = xml_document.xpath(xpath_base_query + '/cvss:availability-impact')
      cvss_source = nvd_base_url + cve['CVE']
      cvss_date = xml_document.xpath(xpath_base_query + '/cvss:generated-on-datetime')

      #Transfer information to YML file
      compiled_cvss = Hash.new
      compiled_cvss['score'] = cvss_score.text.to_f
      compiled_cvss['confidentiality'] = confidentiality_impact.text.downcase
      compiled_cvss['integrity'] = integrity_impact.text.downcase
      compiled_cvss['availability'] = availability_impact.text.downcase
      compiled_cvss['access_complexity'] = access_complexity.text.downcase
      compiled_cvss['authentication'] = authentication.text.downcase
      compiled_cvss['gained_access'] = access_vector.text.downcase
      compiled_cvss['source'] = cvss_source
      compiled_cvss['date_generated'] = cvss_date.text
      cve['cvss'] = compiled_cvss

      #Write modified YML file
      File.open(cve_dir + '/' + cve['CVE'] + '.yml', 'w') {|f| f.write cve.to_yaml}

    end
  end

  #delete XMLs since they're no longer needed
  File.delete(xml_dir)
end
