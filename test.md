``` yaml
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
