``` yaml
---
---
- name: Run Python script from scripts folder on localhost
  hosts: localhost
  connection: local
  vars:
    username: "your_username"
    password: "your_password"
    script_folder: "scripts"  # Folder where the Python script is located
    script_name: "your_script.py"  # Python script filename

  tasks:
    - name: Check if the Python script exists in the scripts folder
      ansible.builtin.stat:
        path: "{{ playbook_dir }}/{{ script_folder }}/{{ script_name }}"
      register: script_check

    - name: Fail if the script does not exist
      fail:
        msg: "The Python script '{{ script_name }}' was not found in the '{{ script_folder }}' folder."
      when: not script_check.stat.exists

    - name: Run Python script with arguments from the scripts folder
      ansible.builtin.command:
        cmd: "python3 {{ script_name }} {{ username }} {{ password }}"
        chdir: "{{ playbook_dir }}/{{ script_folder }}"  # Change directory to the scripts folder
      register: result

    - name: Show script output
      debug:
        var: result.stdout

    - name: Show script error (if any)
      debug:
        var: result.stderr
        when: result.rc != 0

---
- name: Run Python script with username and password arguments
  hosts: all
  vars:
    username: "your_username"
    password: "your_password"
    script_path: "/path/to/your_script.py"  # Update the path of the script on the target machine

  tasks:
    - name: Copy Python script to remote host (if required)
      ansible.builtin.copy:
        src: ./your_script.py  # Path to your Python script on the control node
        dest: "{{ script_path }}"
        mode: '0755'
      when: not ansible.builtin.stat(path=script_path).stat.exists

    - name: Run Python script with arguments
      ansible.builtin.command:
        cmd: "python3 {{ script_path }} {{ username }} {{ password }}"
      register: result

    - name: Show script output
      debug:
        var: result.stdout

    - name: Show script error (if any)
      debug:
        var: result.stderr
        when: result.rc != 0

```

``` groovy
def generateTextBoxHtml(type) {
    def html
    switch (type) {
        case 'small':
            html = '<input type="text" style="width: 100px;" placeholder="Small Text Box"/>'
            break
        case 'medium':
            html = '<input type="text" style="width: 200px;" placeholder="Medium Text Box"/>'
            break
        case 'large':
            html = '<input type="text" style="width: 300px;" placeholder="Large Text Box"/>'
            break
        default:
            html = '<input type="text" style="width: 150px;" placeholder="Default Text Box"/>'
            break
    }
    return html
}

// Example usage
def textBoxHtml = generateTextBoxHtml('medium')
println textBoxHtml

```
