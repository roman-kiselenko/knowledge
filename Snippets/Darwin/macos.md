#### Remove docker on Mac Os

```bash
sudo rm -Rf /Applications/Docker.app
sudo rm -f /usr/local/bin/docker
sudo rm -f /usr/local/bin/docker-machine
sudo rm -f /usr/local/bin/com.docker.cli
sudo rm -f /usr/local/bin/docker-compose
sudo rm -f /usr/local/bin/docker-compose-v1
sudo rm -f /usr/local/bin/docker-credential-desktop
sudo rm -f /usr/local/bin/docker-credential-ecr-login
sudo rm -f /usr/local/bin/docker-credential-osxkeychain
sudo rm -f /usr/local/bin/hub-tool
sudo rm -f /usr/local/bin/hyperkit
sudo rm -f /usr/local/bin/kubectl.docker
sudo rm -f /usr/local/bin/vpnkit
sudo rm -Rf ~/.docker
sudo rm -Rf ~/Library/Containers/com.docker.docker
sudo rm -Rf ~/Library/Application\ Support/Docker\ Desktop
sudo rm -Rf ~/Library/Group\ Containers/group.com.docker
sudo rm -f ~/Library/HTTPStorages/com.docker.docker.binarycookies
sudo rm -f /Library/PrivilegedHelperTools/com.docker.vmnetd
sudo rm -f /Library/LaunchDaemons/com.docker.vmnetd.plist
sudo rm -Rf ~/Library/Logs/Docker\ Desktop
sudo rm -Rf /usr/local/lib/docker
sudo rm -f ~/Library/Preferences/com.docker.docker.plist
sudo rm -Rf ~/Library/Saved\ Application\ State/com.electron.docker-frontend.savedState
sudo rm -f ~/Library/Preferences/com.electron.docker-frontend.plist
rm -rf /opt/homebrew/share/zsh/site-functions/_docker
rm -rf /opt/homebrew/etc/bash_completion.d/docker
rm -rf /opt/homebrew/share/fish/vendor_completions.d/docker.fish
rm -rf /opt/homebrew/share/zsh/site-functions/_docker
```

#### Mac OS terminal 
TIL macOS users can run `sudo spctl developer-mode enable-terminal` to add the Developer Tools section to their Security/Privacy settings. Apps added to this list skip notarization.

#### Brew clean cache
```shell
brew cleanup --prune=all
```

#### MacOS Keyboard Shortcuts

| Combination | Description |
|---|---|
| Fn-Shift-A | Opens Launchpad |
| Fn-C | Opens Control Center |
|Fn-D| Starts dictation (or set a modifier key to do this when you press it twice) |
|Fn-E| Open the emoji picker (same as choosing Edit > Emoji & Symbols) |
|Fn-F| Toggles full-screen mode |
|Fn-H| Hides current windows to reveal the desktop; a second press restores them |
|Fn-M| Selects the Apple menu, after which you can use the arrow keys to navigate menus and activate the selected command by pressing Return |
|Fn-N| Displays Notification Center |
|Fn-Q| Starts a new Quick Note in Notes |
|Fn-Delete| Forward delete on keyboards without a Forward Delete key (or use Control-D) |
|Fn-Up Arrow| Scroll up one page (same as the Page Up key) |
|Fn-Down Arrow| Scroll down one page (same as the Page Down key) |
|Fn-Left Arrow| Scroll to the beginning of a document (same as the Home key) |
|Fn-Right Arrow| Scroll to the end of a document (same as the End key) |

#### Move window with mouse 

```shell
defaults write -g NSWindowShouldDragOnGesture -bool true
defaults write -g NSWindowShouldDragOnGestureFeedback -bool false
```