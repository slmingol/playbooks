# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = '2'

# Adds N boxes using the base name specified
def add_boxes( name, number, properties = {} )
  raise "Number of boxes #{number} is not positive" if number < 1
  if number == 1
    VB_BOXES[ name ] = properties
  else
    ( 1 .. number ).each{ |j| VB_BOXES[ "#{ name }#{ j }" ] = properties }
  end
end


CPUS               = 2
MEMORY             = 1024
VAGRANT_DOMAIN     = 'vm'
DNS_PORT           = 53
MYSQL_PORT         = 3306
ZOOKEEPER_PORT     = 2181
ETCD_PORT          = 4001
HELIOS_MASTER_PORT = 5801
WEB_PORT           = 8080
VERBOSE            = '' # Ansible verbosity level: '', 'v', 'vv', 'vvv', 'vvvv'
REPO_IMPORT        = 'https://s3-eu-west-1.amazonaws.com/evgenyg-ansible/repo-import.zip'
HELIOS_PROPERTIES  = { helios_master: "helios-master.#{ VAGRANT_DOMAIN }",
                       domain:        VAGRANT_DOMAIN,
                       use_consul:    false }
HELIOS_PORTS       = [ DNS_PORT, ZOOKEEPER_PORT, ETCD_PORT, HELIOS_MASTER_PORT ]

VB_BOXES = {
  jvm:     {},
  packer:  {},
  mysql:   { vagrant_ports: [ MYSQL_PORT ]},
  jenkins: { vagrant_ports: [ WEB_PORT ]},
  helios:  HELIOS_PROPERTIES.merge( vagrant_ports: HELIOS_PORTS,
                                    helios_master: "helios.#{ VAGRANT_DOMAIN }" ),
  repo:          { memory:          2048, # For Artifactory MySQL
                   port:            WEB_PORT,
                   import:          REPO_IMPORT,
                   vagrant_ports:   [ WEB_PORT ],
                   playbook:        'artifactory' },
                  #  playbook:        'nexus' },
  'test-repo' => { reports_dir:     '/opt/gatling-reports',
                   reports_archive: '/vagrant/gatling-reports.tar.gz',
                   upload:          REPO_IMPORT,
                   clean_reports:   true,
                   run_simulations: false,
                   host:            "repo.#{ VAGRANT_DOMAIN }",
                   port:            WEB_PORT,
                   repo_name:       'Artifactory' }
                  #  repo_name:       'Nexus' }
}

add_boxes( 'zookeeper',     3, zk_instances: (1..3).map{ |j| "zookeeper#{ j }.#{ VAGRANT_DOMAIN }" } )
add_boxes( 'helios-master', 2, HELIOS_PROPERTIES.merge( vagrant_ports: HELIOS_PORTS ))
add_boxes( 'helios-agent',  2, HELIOS_PROPERTIES.merge( vagrant_ports: [ WEB_PORT ] ))

Vagrant.require_version '>= 1.7.0'
Vagrant.configure( VAGRANTFILE_API_VERSION ) do | config |

  # https://github.com/phinze/landrush
  # vagrant plugin install landrush
  # vagrant landrush start|stop|restart|status|ls|vms|help
  # ~/.vagrant.d/data/landrush
  config.landrush.enabled = true
  config.landrush.tld     = VAGRANT_DOMAIN

  VB_BOXES.each_pair { | box, variables |

    box_name = "#{ box }.#{ VAGRANT_DOMAIN }"

    config.vm.define box do | b |
      b.vm.box              = 'ubuntu/trusty64'
      b.vm.box_check_update = true
      b.vm.hostname         = box_name # 1) Vagrant will cut out everything starting from the first dot but ..
                                       # 2) Landrush needs a proper hostname ending with config.landrush.tld
      b.vm.synced_folder 'playbooks', '/playbooks'

      ( variables[ :vagrant_ports ] || [] ).each { | port |
        # https://docs.vagrantup.com/v2/networking/forwarded_ports.html
        b.vm.network 'forwarded_port', guest: port, host: port, auto_correct: true, protocol: 'tcp'
      }

      ( variables[ :vagrant_ports_udp ] || [] ).each { | port |
        # https://docs.vagrantup.com/v2/networking/forwarded_ports.html
        b.vm.network 'forwarded_port', guest: port, host: port, auto_correct: true, protocol: 'udp'
      }

      b.vm.provider :virtualbox do | vb |
        vb.gui  = false
        vb.name = box_name
        # https://www.virtualbox.org/manual/ch08.html
        # http://superuser.com/a/883328/239763
        vb.customize [ 'modifyvm', :id,
                       '--natdnshostresolver1', 'on',
                       '--memory', variables[ :memory ] || MEMORY,
                       '--cpus',   variables[ :cpus ]   || CPUS ]
      end

      b.vm.provision :ansible do | ansible |
        ansible.verbose    = VERBOSE if VERBOSE != ''
        ansible.playbook   = "playbooks/#{ variables[ :playbook ] || "#{ box.to_s.gsub( /\d+$/, '' ) }" }-ubuntu.yml"
        ansible.extra_vars = variables.merge({
          # Uncomment and set to true to forcefully update all packages
          # Uncomment and set to false to disable periodic run
          # Otherwise (when commented out) packages are updated automatically once a day
          # periodic: true,
          # periodic: false,

          # Uncomment to simulate a Docker container behavior (less packages are installed)
          # is_docker: true
        })
      end
    end
  }
end
