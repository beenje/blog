.. title: Installing OpenVPN on a Raspberry Pi with Ansible
.. slug: installing-openvpn-on-a-raspberry-pi-with-ansible
.. date: 2016-07-18 22:26:28 UTC+02:00
.. tags: pi,VPN,Ansible
.. category: linux
.. link:
.. description:
.. type: text

I have to confess that I initially decided to install a VPN,
not to secure my connection when using a free Wireless Acces Point in an
airport or hotel, but to watch Netflix :-)

I had a VPS in France where I installed sniproxy to access Netflix.
Not that I find the french catalogue so great, but as a French guy living
in Sweden, it was a good way for my kids to watch some french programs.
But Netflix started to block VPS providers...

I have a brother in France who has a Fiber Optic Internet access.
That was a good opportunity to setup a private VPN and I bought him a Raspberry Pi.

There are many resources on the web about OpenVPN_.
A paper worth mentioning is: `SOHO Remote Access VPN. Easy as Pie, Raspberry Pi...
<https://www.sans.org/reading-room/whitepapers/networkdevs/soho-remote-access-vpn-easy-pie-raspberry-pi-34427>`_
It's from end of 2013 and describes Esay-RSA 2.0 (that used to be installed with
OpenVPN), but it's still an interesting read.

Anyway, most resources describe all the commands to run.
I don't really like installing softwares by running a bunch of commands. Propably due
to my professional experience, I like things to be reproducible.
That's why I love to automate things. I wrote a lot of shell scripts over
the years. About two years ago, I discovered Ansible_ and it quickly became my
favorite tool to deploy software.

So let's write a small Ansible playbook to install OpenVPN on a Raspberry Pi.

First the firewall configuration. I like to use `ufw
<https://help.ubuntu.com/community/UFW>`_ which is quite easy to
setup::

    - name: install dependencies
      apt: name=ufw state=present update_cache=yes cache_valid_time=3600

    - name: update ufw default forward policy
      lineinfile: dest=/etc/default/ufw regexp=^DEFAULT_FORWARD_POLICY line=DEFAULT_FORWARD_POLICY="ACCEPT"
      notify: reload ufw

    - name: enable ufw ip forward
      lineinfile: dest=/etc/ufw/sysctl.conf regexp=^net/ipv4/ip_forward line=net/ipv4/ip_forward=1
      notify: reload ufw

    - name: add NAT rules to ufw
      blockinfile:
        dest: /etc/ufw/before.rules
        insertbefore: BOF
        block: |
          # Nat table
          *nat
          :POSTROUTING ACCEPT [0:0]

          # Nat rules
          -F
          -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j SNAT --to-source {{ansible_eth0.ipv4.address}}

          # don't delete the 'COMMIT' line or these nat rules won't be processed
          COMMIT
      notify: reload ufw

    - name: allow ssh
      ufw: rule=limit port=ssh proto=tcp

    - name: allow openvpn
      ufw: rule=allow port={{openvpn_port}} proto={{openvpn_protocol}}

    - name: enable ufw
      ufw: logging=on state=enabled

This enables IP forwarding, adds the required NAT rules and allows ssh and
openvpn.

The rest of the playbook installs OpenVPN and generates all the keys automatically,
except the Diffie-Hellman one that should be generated locally.
This is just because it takes for ever on the Pi :-)

::

    - name: install openvpn
      apt: name=openvpn state=present

    - name: create /etc/openvpn
      file: path=/etc/openvpn state=directory mode=0755 owner=root group=root

    - name: create /etc/openvpn/keys
      file: path=/etc/openvpn/keys state=directory mode=0700 owner=root group=root

    - name: create clientside and serverside directories
      file: path="{{item}}" state=directory mode=0755
      with_items:
          - "{{clientside}}/keys"
          - "{{serverside}}"
      become: true
      become_user: "{{user}}"

    - name: create openvpn base client.conf
      template: src=client.conf.j2 dest={{clientside}}/client.conf owner=root group=root mode=0644

    - name: download EasyRSA
      get_url: url={{easyrsa_url}} dest=/home/{{user}}/openvpn
      become: true
      become_user: "{{user}}"

    - name: create scripts
      template: src={{item}}.j2 dest=/home/{{user}}/openvpn/{{item}} owner=root group=root mode=0755
      with_items:
        - create_serverside
        - create_clientside
      tags: client

    - name: run serverside script
      command: ./create_serverside
      args:
        chdir: /home/{{user}}/openvpn
        creates: "{{easyrsa_server}}/ta.key"
      become: true
      become_user: "{{user}}"

    - name: run clientside script
      command: ./create_clientside {{item}}
      args:
        chdir: /home/{{user}}/openvpn
        creates: "{{clientside}}/files/{{item}}.ovpn"
      become: true
      become_user: "{{user}}"
      with_items: "{{openvpn_clients}}"
      tags: client

    - name: install all server keys
      command: install -o root -g root -m 600 {{item.name}} /etc/openvpn/keys/
      args:
        chdir: "{{item.path}}"
        creates: /etc/openvpn/keys/{{item.name}}
      with_items:
        - { name: 'ca.crt', path: "{{easyrsa_server}}/pki" }
        - { name: '{{ansible_hostname}}.crt', path: "{{easyrsa_server}}/pki/issued" }
        - { name: '{{ansible_hostname}}.key', path: "{{easyrsa_server}}/pki/private" }
        - { name: 'ta.key', path: "{{easyrsa_server}}" }

    - name: copy Diffie-Hellman key
      copy: src="{{openvpn_dh}}" dest=/etc/openvpn/keys/dh.pem owner=root group=root mode=0600

    - name: create openvpn server.conf
      template: src=server.conf.j2 dest=/etc/openvpn/server.conf owner=root group=root mode=0644
      notify: restart openvpn

    - name: start openvpn
      service: name=openvpn state=started

The *create_clientside* script generates all the required client keys and creates an ovpn file
that includes them.  It makes it very easy to install on any device: just one file to
drop.

One thing I stumbled upon is the *ns-cert-type server* option that I
initially used in the server configuration. This prevented the client to
connect. As explained `here
<https://community.openvpn.net/openvpn/wiki/EasyRSA3-OpenVPN-Howto>`_,
this option is a deprecated "Netscape" cert attribute. It's not enabled by
default with Easy-RSA 3.

Fortunately, the mentioned `howto
<https://community.openvpn.net/openvpn/wiki/EasyRSA3-OpenVPN-Howto>`_ and
the `Easy-RSA github <https://github.com/OpenVPN/easy-rsa>`_ page are good references
for Easy-RSA 3.

One important thing to note is that I create all the keys with no password.
That's obviously not the most secure and recommended way.
Anyone accessing the CA could sign new requests. But it can be stored offline on an USB stick.
I actually think that for my use case it's not even worth keeping the CA.
Sure it means I can't easily add a new client or revoke a certificate.
But with the playbook, it's super easy to throw all the keys and regenerate everything.
That forces to replace all clients configuration but with 2 or 3
clients, this is not a problem.

For sure don't leave all the generated keys on the Pi!
After copying the clients ovpn files, remove the /home/pi/openvpn
directory (save it somewhere safe if you want to add new clients or revoke
a certificate without regenerating everything).

The full playbook can be found on `github <https://github.com/beenje/pi_openvpn>`_.
The README includes some quick instructions.

I now have a private VPN in France and one at home that I can use to
securely access my NAS from anywhere!

.. _OpenVPN: https://openvpn.net/index.php/open-source/documentation/howto.html
.. _Ansible: http://docs.ansible.com/ansible/index.html
