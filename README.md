name: Generate GitHub Profile Stats

on:
  schedule:
    - cron: '0 0 * * *' # diário
  workflow_dispatch: {}

jobs:
  generate:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Fetch GitHub profile data and generate SVG
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          set -e
          mkdir -p assets
          USER="Thulio-Zago"

          echo "Fetching user data..."
          user_json=$(curl -s "https://api.github.com/users/$USER")
          followers=$(echo "$user_json" | jq -r '.followers')
          public_repos=$(echo "$user_json" | jq -r '.public_repos')

          echo "Calculating total stars (may take a moment)..."
          # pega até 100 repositórios; para mais repos, usar paginação
          stars=$(curl -s "https://api.github.com/users/$USER/repos?per_page=100" | jq '[.[].stargazers_count] | add')
          stars=${stars:-0}

          echo "Gerando SVG em assets/github-stats.svg"
          cat > assets/github-stats.svg <<'SVG'
<svg xmlns="http://www.w3.org/2000/svg" width="680" height="140">
  <rect width="100%" height="100%" fill="#0f172a"/>
  <g fill="#cbd5e1" font-family="system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial" font-size="16">
    <text x="24" y="42">Repositórios: REPOS</text>
    <text x="24" y="72">Seguidores: FOLLOWERS</text>
    <text x="24" y="102">Estrelas totais: STARS</text>
  </g>
</svg>
SVG

          sed -i "s/REPOS/${public_repos}/g" assets/github-stats.svg
          sed -i "s/FOLLOWERS/${followers}/g" assets/github-stats.svg
          sed -i "s/STARS/${stars}/g" assets/github-stats.svg

      - name: Commit and push if changed
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add assets/github-stats.svg || true
          if ! git diff --staged --quiet; then
            git commit -m "chore: update github stats svg"
            git push
          else
            echo "Sem alterações para commitar"
          fi
