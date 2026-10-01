# Zsh + Oh My Zsh + DevOps Toolkit Setup (Runbook)

Covers: zsh, Oh My Zsh, lolcat, figlet, DevOps plugins, network and system tools, k6, Grafana, daily aliases, and suggested extra tools.

> Run all commands yourself on the target machine. Confirm the environment first (laptop, staging, or prod bastion). Do not run this on a production host without approval.

---

## Prerequisites

```bash
git --version && curl --version | head -1
[ -f ~/.zshrc ] && cp ~/.zshrc ~/.zshrc.bak.$(date +%Y%m%d-%H%M%S)
```

---

## 1. Install base packages (pick your OS)

### Debian/Ubuntu
```bash
sudo apt update
sudo apt install -y zsh lolcat figlet fzf jq tmux
```

### Fedora/RHEL
```bash
sudo dnf install -y zsh figlet fzf jq tmux ruby rubygems
sudo gem install lolcat   # lolcat is not in the default repos
```

### Arch
```bash
sudo pacman -S zsh lolcat figlet fzf jq tmux
```

### macOS (Homebrew)
```bash
brew install zsh lolcat figlet fzf jq tmux
```

### Optional: make zsh the default shell
Log out and back in afterwards. If `chsh` rejects the path, make sure it is listed in `/etc/shells`.
```bash
chsh -s "$(which zsh)"
```

---

## 2. Install Oh My Zsh

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

The installer backs up an existing `~/.zshrc` to `~/.zshrc.pre-oh-my-zsh`.

---

## 3. Install external plugins

```bash
ZSH_CUSTOM=${ZSH_CUSTOM:-~/.oh-my-zsh/custom}
git clone https://github.com/zsh-users/zsh-autosuggestions     $ZSH_CUSTOM/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting $ZSH_CUSTOM/plugins/zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-completions         $ZSH_CUSTOM/plugins/zsh-completions
git clone https://github.com/Aloxaf/fzf-tab                    $ZSH_CUSTOM/plugins/fzf-tab
```

If a directory already exists, git errors out. That is harmless, so skip it.

---

## 4. Configure ~/.zshrc

```bash
vim ~/.zshrc
```

Add the `fpath` line and set `plugins` **before** the `source $ZSH/oh-my-zsh.sh` line:

```bash
fpath+=${ZSH_CUSTOM:-${ZSH:-~/.oh-my-zsh}/custom}/plugins/zsh-completions/src

plugins=(
  git gh
  docker docker-compose podman
  kubectl kubectx kube-ps1 helm minikube kind argocd flux eksctl
  terraform ansible
  aws gcloud azure doctl
  tmux fzf jsontools sudo colored-man-pages
  zsh-completions
  fzf-tab
  zsh-autosuggestions
  zsh-syntax-highlighting   # keep last
)

source $ZSH/oh-my-zsh.sh
```

Add this **after** the `source` line (otherwise Oh My Zsh overwrites the prompt):

```bash
KUBE_PS1_SYMBOL_ENABLE=false
PROMPT='$(kube_ps1) '$PROMPT

# Optional banner (cosmetic, adds startup time)
# figlet "$(hostname -s)" | lolcat

# Optional: activate zoxide and direnv (after installing them in section 6)
# eval "$(zoxide init zsh)"
# eval "$(direnv hook zsh)"
```

---

## 5. Reload and verify

```bash
source ~/.zshrc
figlet "zsh ready" | lolcat
echo $plugins
alias | grep -E '^(k|kgp|tf|tfa)='
type _kubectl >/dev/null && echo "kubectl completion OK"
time zsh -i -c exit   # startup should be under ~0.5s
```

If startup is slow, remove cloud plugins you do not use (`azure`, `gcloud`, `doctl`, etc.).

---

## 6. Install DevOps tools

`netstat` is part of the `net-tools` package. `net-tools` is deprecated on modern Linux, so `iproute2` is installed too for its replacement, `ss`.

### Debian/Ubuntu: base tools
```bash
sudo apt update
sudo apt install -y net-tools iproute2 tree tldr htop btop ncdu \
  dnsutils traceroute mtr-tiny nmap tcpdump netcat-openbsd lsof strace sysstat \
  ripgrep fd-find bat unzip httpie
tldr --update    # fetch the tldr page cache (some versions need this)
```
On Debian/Ubuntu, `bat` installs as `batcat` and `fd` as `fdfind`. The alias file in section 7 fixes that.

### macOS
```bash
brew install tree tldr htop btop ncdu mtr nmap ripgrep fd bat httpie k6 grafana
# net-tools is Linux-only. macOS already ships netstat, and lsof/nettop are built in.
```

### Fedora
```bash
sudo dnf install -y net-tools iproute tree htop btop ncdu bind-utils mtr nmap tcpdump nmap-ncat lsof ripgrep fd-find bat
```

### Arch
```bash
sudo pacman -S net-tools iproute2 tree tldr htop btop ncdu bind mtr nmap tcpdump openbsd-netcat lsof ripgrep fd bat
```

### k6 (Debian/Ubuntu)
```bash
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
  --keyserver hkp://keyserver.ubuntu.com:80 \
  --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" \
  | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt update && sudo apt install -y k6
k6 version
```

### Grafana server (Debian/Ubuntu)
```bash
sudo apt install -y apt-transport-https software-properties-common wget gpg
sudo mkdir -p /etc/apt/keyrings
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" \
  | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update && sudo apt install -y grafana
sudo systemctl enable --now grafana-server
systemctl status grafana-server --no-pager
```

- Grafana listens on `http://localhost:3000`.
- Default login is `admin` / `admin`. Change the password on first login and do not expose port 3000 publicly.
- Repo keys and URLs occasionally change. If a step fails, check the official k6 and Grafana install docs.
- For Grafana on Kubernetes, use Helm: `helm repo add grafana https://grafana.github.io/helm-charts`

---

## 7. Daily DevOps aliases and functions

Oh My Zsh loads every `*.zsh` file in `custom/` automatically, which keeps `.zshrc` clean.

```bash
vim ~/.oh-my-zsh/custom/devops.zsh
```

```zsh
# ---------- Safety: remove risky plugin shortcuts ----------
unalias tfd 2>/dev/null          # terraform destroy
unalias tfa 2>/dev/null          # terraform apply (use reviewed plan instead)

# ---------- Prod guard ----------
_is_prod() { [[ "$(kubectl config current-context 2>/dev/null)" == *prod* ]]; }
kguard() {
  if _is_prod; then
    read -q "REPLY?PROD context ($(kubectl config current-context)). Continue? [y/N] " || { echo; return 1; }
    echo
  fi
  command kubectl "$@"
}
alias kdel='kguard delete'
alias kscale='kguard scale'
alias kapply='kguard apply'
alias krr='kguard rollout restart'

# ---------- Kubernetes ----------
alias kctxs='kubectl config get-contexts'
alias kwho='kubectl config current-context'
alias kns='kubectl config view --minify -o jsonpath="{..namespace}"; echo'
alias kbad='kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded'
alias kev='kubectl get events -A --sort-by=.lastTimestamp | tail -30'
alias ktp='kubectl top pods -A --sort-by=memory | head -20'
alias ktn='kubectl top nodes'
alias krs='kubectl rollout status'
alias krh='kubectl rollout history'
alias kimg='kubectl get pods -A -o jsonpath="{range .items[*]}{.metadata.namespace}{\"\t\"}{.metadata.name}{\"\t\"}{.spec.containers[*].image}{\"\n\"}{end}"'
ksh()   { kubectl exec -it "$1" -- sh -c 'bash || sh'; }
klogs() { kubectl logs -f --tail=100 "$@"; }

# ---------- Terraform (plan first, apply only the reviewed plan) ----------
alias tfpl='terraform plan -out=tfplan'
alias tfsh='terraform show tfplan'
alias tfap='terraform apply tfplan'
alias tffmt='terraform fmt -recursive'
alias tfval='terraform init -backend=false && terraform validate'

# ---------- Docker ----------
alias dps='docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"'
alias dlf='docker logs -f --tail 100'
alias dstat='docker stats --no-stream'
alias dprune='docker system prune'   # prompts before deleting

# ---------- Helm / AWS ----------
alias hls='helm ls -A'
alias awho='aws sts get-caller-identity'

# ---------- System and network ----------
alias ll='ls -lah'
alias ports='sudo ss -tulpn'
alias netl='sudo netstat -tulpn'
alias myip='curl -s ifconfig.me; echo'
alias dfh='df -hT'
alias psmem='ps aux --sort=-%mem | head -15'
alias pscpu='ps aux --sort=-%cpu | head -15'
command -v batcat >/dev/null && alias bat='batcat'
command -v fdfind >/dev/null && alias fd='fdfind'

portcheck() { nc -zv -w 3 "$1" "$2"; }     # portcheck host 443
certexp()   { echo | openssl s_client -servername "$1" -connect "$1:443" 2>/dev/null | openssl x509 -noout -subject -dates; }
mkcd()      { mkdir -p "$1" && cd "$1"; }
```

The `kguard` wrapper prompts for confirmation whenever the current kube context name contains `prod`. Adjust `*prod*` if your context names differ. The `tfpl` and `tfap` aliases force a review-then-apply flow.

### Reload and verify
```bash
source ~/.zshrc
type kguard kbad tfpl ports
tldr tar
k6 version && tree -L 1 ~
```

### k6 quick smoke test
```bash
cat > smoke.js <<'EOF'
import http from 'k6/http';
import { check, sleep } from 'k6';
export const options = { vus: 5, duration: '30s' };
export default function () {
  const r = http.get('https://test.k6.io');
  check(r, { 'status is 200': (x) => x.status === 200 });
  sleep(1);
}
EOF
k6 run smoke.js
```
Only run load tests against systems you own. In production, only with approval.

---

## 8. Suggested additional tools

| Category | Tools | Why |
|---|---|---|
| Kubernetes UX | **k9s**, **kubectx/kubens**, **stern**, **kubecolor** | Terminal UI, fast context and namespace switching, multi-pod log tailing |
| Kubernetes quality | **kubeconform**, **kustomize**, **helm-diff**, **kube-linter** | Validate manifests and preview Helm changes before apply |
| IaC | **tflint**, **terragrunt**, **infracost**, **checkov** or **tfsec**, **pre-commit** | Linting, cost estimates, security scans, commit-time checks |
| Security | **trivy**, **sops + age**, **vault CLI**, **gitleaks** | Image and IaC scans, encrypted secrets, secret leak detection |
| Containers | **lazydocker**, **dive**, **ctop** | Inspect containers and image layers |
| Data wrangling | **yq**, **jq**, **fx** | Edit YAML and JSON from the CLI |
| Shell productivity | **zoxide**, **direnv**, **eza**, **atuin** | Smarter `cd`, per-directory env vars, better `ls` and history |
| Observability | **promtool**, **amtool**, **logcli** (Loki) | Validate Prometheus rules, manage alerts, query logs |
| Load testing | **hey**, **vegeta**, **wrk** | Lightweight alternatives to k6 |
| Cloud CLIs | **awscli v2**, **gcloud**, **eksctl**, **aws-vault** | Cloud management and safer credential handling |

### Quick installs
```bash
# macOS
brew install k9s kubectx stern kubecolor kustomize tflint trivy sops age yq zoxide direnv eza lazydocker dive

# Debian/Ubuntu: k9s, stern, trivy, tflint, sops and yq are best installed from their
# GitHub releases or official repos, because the apt versions are missing or outdated.
sudo apt install -y kubectx direnv zoxide
```

---

## Usage tips

- Use a figlet banner as a prod warning, for example `figlet "PROD" | lolcat`, ideally inside a function that runs when your kube context matches production.
- Always use `tfpl` then `tfsh` then `tfap` in production, and include the plan output in your PR.

---

## Rollback

```bash
# Restore previous zshrc and remove external plugins
cp ~/.zshrc.bak.<timestamp> ~/.zshrc && source ~/.zshrc
rm -rf ~/.oh-my-zsh/custom/plugins/{zsh-autosuggestions,zsh-syntax-highlighting,zsh-completions,fzf-tab}

# Remove the alias file
rm ~/.oh-my-zsh/custom/devops.zsh && source ~/.zshrc

# Remove Grafana and k6
sudo systemctl disable --now grafana-server
sudo apt remove -y grafana k6
sudo rm -f /etc/apt/sources.list.d/{grafana,k6}.list

# Remove Oh My Zsh entirely and switch back to bash
uninstall_oh_my_zsh
chsh -s "$(which bash)"
```
