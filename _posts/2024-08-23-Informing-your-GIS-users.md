---
title: "Informing your GIS users"
categories:
    - gis
tags:
    - gis
    - smallworld
    - powershell
---

Were you ever in need to inform all GIS users? How did you do this?

Did you sent an email? To whom?

To all users you know? Do you know all users?

To certain key users with the request to forward the information? Did they do that? Were they at work or did they had a deputy?

Because this way is prone to fail one of my customers had a solution that prompted the users a Magik frame with the necessary information. But without going into detail the solution was quite bad to use and to maintain. 

After the customer went to 5.x the solution revealed another drawback. The information was displayed after the image/session had started. The bad thing in my eyes was the fact that sometimes the image/session automatically quitted after the information because hard datamodel changes or else were made. Imagine a user who waited the standard 5.x start time for his session to be ready only to be informed that it will quit now because of this or that.

I thought I was clever and directed all links for starting a session to a single point of contact batch file. Inside the file I could control the information display and suppress the session start if necessary. I simply wrote an html file and used "start %lt;PATH TO HTML%gt;" to open it in a web browser. What I didn't knew until then: the Citrix workers didn't knew how to open it. And of course there were users who used a link that I couldn't redirect anymore to my single point of contact.

So I returned to a Magik solution. I read about SWIFT and fdoc_abstract_document_element, formatted_document, html5_component and nearly died trying to find out how to work with them. When I had a working solution that fully supports html using html5_component I put it in a procedure that should be executed prior loading any other Magik code like sessions. I started the session and... it failed! The exemplar html5_component isn't known to the system at this point and it even isn't able to load its smallworld module because it wasn't known, too.

Yes, I was frustrated.

But I knew another way I should try: using Powershell. Because of [Mark's post](https://markhing.com/smallworld/using-powershell-with-smallworld-gis-magik/) the Magik side was not a great deal.

```
inform_user <<
	_proc@gw_inform_user(p_display_information?, p_quit_session?)
		_if p_display_information?
		_then
			_local l_powershell << "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe"
			_local l_execution_policy << "-ExecutionPolicy Unrestricted"
			_local l_command << "-Command <PATH>\CUSTOMER_PRODUCTS\config\magik_sessions\resources\powershell\inform_user.ps1"
			_local l_powershell_command << write_string_with_separator(
									{l_powershell,
								l_execution_policy,
								l_command},
									% )

			_local l_cmd_result << system.input_from_command(l_powershell_command)
			_loop
				_if (l_line << l_cmd_result.get_line()) _is _unset _then _leave _endif
				show(l_line)
			_endloop
		_endif

		_if p_quit_session?
		_then quit()
		_endif
		
	_endproc
```

For creating an UI with Powershell I used chatGPT. It took a few attempts to generate this code

```powershell
Add-Type -AssemblyName System.Windows.Forms

# Create a form
$form = New-Object System.Windows.Forms.Form
$form.Text = "BlaBlaBla"
$form.Size = New-Object System.Drawing.Size(600,400)
$form.Icon = New-Object system.drawing.icon ("$PSScriptRoot\..\base\bitmaps\sw_target.ico")

# Create a WebBrowser control to display HTML content
$webBrowser = New-Object System.Windows.Forms.WebBrowser
$webBrowser.Dock = 'Fill'

# HTML content to be displayed
$htmlContent = Get-Content -Path "$PSScriptRoot\..\html\index.html" -Encoding utf8 -Raw

# Load HTML content directly into the WebBrowser control
$webBrowser.DocumentText = $htmlContent

# Create a panel for the button
$buttonPanel = New-Object System.Windows.Forms.Panel
$buttonPanel.Dock = 'Bottom'
$buttonPanel.Height = 50  

# Create a button
$button = New-Object System.Windows.Forms.Button
$button.Text = "Ok"
$button.Height = 40

$buttonPanel.Controls.Add($button)
$button.Add_Click({$form.Close()})

# Add controls to the form
$form.Controls.Add($webBrowser)
$form.Controls.Add($buttonPanel)

# Show the form
$form.Add_Shown({ $form.Activate() })
[void]$form.ShowDialog()

Write-Output "information read"
```

But now we have a useful and convenient solution for informing our users. Because it uses html we're able to include more features than displaying text if we want to.