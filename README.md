# VPS Audit Ansible Playbook

A simple Ansible playbook to run security audits on multiple VPS servers remotely.

## Requirements

* Ansible installed on the control node
* SSH access to target servers
* Sudo privileges on target servers
* VPS audit bash script (`vps-audit.sh`)

## Setup

1. Clone or download this repository
2. Place the `vps-audit.sh` script in the playbook directory
3. Create your inventory file
4. Ensure SSH access to your target servers

## Directory Structure
```
.
├── vps_audit.yml          # Ansible playbook
├── inventory.ini          # Server inventory
├── vps-audit.sh          # Audit script
└── audit_reports/        # Generated reports
```

## Configuration

### Example Inventory File (inventory.ini)
```ini
[servers]
server1 ansible_host=192.168.1.10
server2 ansible_host=192.168.1.11
server3 ansible_host=example.com
```

### Example Playbook (vps_audit.yml)
```yaml
---
- name: Run VPS Audit Script
  hosts: all
  become: yes
  vars:
    script_path: "/tmp/vps-audit.sh"
    local_reports_dir: "./audit_reports"
    timestamp: "{{ ansible_date_time.year }}{{ ansible_date_time.month }}{{ ansible_date_time.day }}_{{ ansible_date_time.hour }}{{ ansible_date_time.minute }}{{ ansible_date_time.second }}"

  tasks:
    - name: Create local reports directory with proper permissions
      local_action:
        module: file
        path: "{{ local_reports_dir }}"
        state: directory
        mode: '0755'
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
      run_once: true

    - name: Copy VPS audit script to remote hosts
      copy:
        src: "./vps-audit.sh"
        dest: "{{ script_path }}"
        mode: '0755'

    - name: Execute VPS audit script
      shell: "{{ script_path }}"
      args:
        chdir: /tmp
      register: audit_result

    - name: Find the generated report file
      find:
        paths: /tmp
        patterns: "vps-audit-report-*.txt"
        age: -1m
      register: report_files

    - name: Fetch audit report
      fetch:
        src: "{{ item.path }}"
        dest: "{{ local_reports_dir }}/{{ inventory_hostname }}-{{ timestamp }}.txt"
        flat: yes
        mode: '0644'
      with_items: "{{ report_files.files }}"
      when: report_files.matched > 0

    - name: Clean up
      file:
        path: "{{ item }}"
        state: absent
      with_items:
        - "{{ script_path }}"
        - "{{ report_files.files | map(attribute='path') | list }}"
      when: report_files.matched > 0
```

## Usage

Run the playbook:
```bash
ansible-playbook -i inventory.ini vps_audit.yml
```

For verbose output:
```bash
ansible-playbook -i inventory.ini vps_audit.yml -vv
```

## Reports

Reports are stored in `audit_reports/` with the format:
`[hostname]-[timestamp].txt`

## Troubleshooting

If you encounter permission issues:
```bash
# Remove existing audit_reports directory
rm -rf audit_reports

# Run the playbook again
ansible-playbook -i inventory.ini vps_audit.yml
```

## Security Notes

- The script runs with elevated privileges (sudo)
- Temporary files are cleaned up after execution
- Reports are stored locally on the control machine
- No sensitive information is left on remote servers

## License

MIT
## Credits

Original VPS audit script by [Binyam Hailemeskel](https://github.com/vernu)  
Source: https://github.com/vernu/vps-audit

## Contributing

Feel free to submit issues and enhancement requests!
