# Commands
```
/opt/CrowdStrike/falconctl -g --rfm-state
/opt/CrowdStrike/falconctl -g --aid
systemctl is-enabled falcon-sensor
systemctl is-active falcon-sensor
```

# Playbook
```
---
- name: Configure CrowdStrike Falcon Sensor
  hosts: falcon_hosts
  become: true
  gather_facts: true

  vars:
    falcon_service: falcon-sensor
    falcon_proxy: "http://proxy.example.com:8080"

  tasks:

    - name: Check Falcon Sensor service state
      ansible.builtin.systemd:
        name: "{{ falcon_service }}"
      register: falcon_service_status
      changed_when: false
      failed_when: false

    - name: Check Falcon Sensor connection
      ansible.builtin.command:
        cmd: /opt/CrowdStrike/falconctl -g --aid
      register: falcon_aid
      changed_when: false
      failed_when: false

    - name: Check Falcon Sensor connection status
      ansible.builtin.command:
        cmd: /opt/CrowdStrike/falconctl -g --rfm-state
      register: falcon_rfm
      changed_when: false
      failed_when: false

    - name: Determine whether Falcon is already healthy
      ansible.builtin.set_fact:
        falcon_healthy: >-
          {{
            falcon_service_status.status.ActiveState == 'active'
            and falcon_service_status.status.UnitFileState == 'enabled'
            and falcon_aid.rc == 0
            and falcon_rfm.rc == 0
            and 'rfm-state=false' in falcon_rfm.stdout
          }}

    - name: Configure Falcon Sensor proxy
      ansible.builtin.command:
        cmd: "/opt/CrowdStrike/falconctl -s --aph={{ falcon_proxy }}"
      when: not falcon_healthy
      notify: Restart Falcon Sensor

  handlers:

    - name: Restart Falcon Sensor
      ansible.builtin.systemd:
        name: "{{ falcon_service }}"
        state: restarted
        enabled: true
        daemon_reload: true
```
