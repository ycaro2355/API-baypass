# bypass.py  (discord.py 2.x, Python 3.8+)
# Instala sozinho o discord.py e o aiohttp na primeira execução.
import os
import sys
import time
import subprocess
import importlib


def ensure(pip_name, module_name):
    """Instala o pacote automaticamente se não estiver instalado."""
    try:
        importlib.import_module(module_name)
    except ImportError:
        print(f"Instalando {pip_name}...")
        subprocess.check_call([sys.executable, "-m", "pip", "install", "-U", pip_name])
        importlib.invalidate_caches()


ensure("discord.py", "discord")
ensure("aiohttp", "aiohttp")

import aiohttp
import discord
from discord import app_commands
from discord.ext import commands

try:
    from dotenv import load_dotenv
    load_dotenv()
except ImportError:
    pass

# Se não usar .env, coloque direto aqui (e não compartilhe o arquivo):
DISCORD_TOKEN = os.getenv("DISCORD_TOKEN") or "COLE_O_TOKEN_DO_BOT_AQUI"

API = "https://api.baconbypass.xyz"
KEY = os.getenv("BACON_API_KEY") or "Bacon-9c95ce9a12fba47abf1b-0c90489c59f6a1f6cc0d"

GREEN = 0x00FF00
RED = 0xFF0000


def error_embed(title: str, description: str) -> discord.Embed:
    return discord.Embed(title=title, description=description, color=RED)


def valid_url(url: str) -> bool:
    url = url.strip()
    return url.startswith(("http://", "https://")) and "." in url and " " not in url


def is_lootlabs(url: str) -> bool:
    u = url.lower()
    return any(d in u for d in ("lootlabs.gg", "loot-link", "lootdest", "loot-labs"))


class CopyView(discord.ui.View):
    """Botão Copy: o bot não acessa a área de transferência, então ele
    manda o resultado só para quem clicou, pronto para copiar."""

    def __init__(self, result: str):
        super().__init__(timeout=600)
        self.result = result

    @discord.ui.button(label="Copy", style=discord.ButtonStyle.success)
    async def copy(self, interaction: discord.Interaction, button: discord.ui.Button):
        await interaction.response.send_message(self.result, ephemeral=True)


async def get_remaining(session: aiohttp.ClientSession):
    """Retorna quantos requests ainda restam, ou None se não der para saber."""
    try:
        async with session.get(f"{API}/check-expired", params={"key": KEY}) as resp:
            d = await resp.json(content_type=None)
        if d.get("status") == "success":
            if "remaining" in d:
                return d["remaining"]
            return int(d["limitRequest"]) - int(d["usedRequests"])
    except Exception:
        pass
    return None


class Bypass(commands.Cog):
    def __init__(self, bot: commands.Bot):
        self.bot = bot

    @app_commands.command(name="bypass", description="Faz bypass de um link suportado")
    @app_commands.describe(url="Link para bypass")
    async def bypass(self, interaction: discord.Interaction, url: str):
        # Mostra "Bacon Bypass está pensando..." enquanto processa
        try:
            await interaction.response.defer()
        except discord.NotFound:
            return  # a interação já expirou (bot demorou/estava fora do ar)

        url = url.strip()
        if not valid_url(url):
            return await interaction.followup.send(
                embed=error_embed("Invalid URL", "Please provide a valid URL.")
            )

        # LootLabs: responde com o link do site de bypass (igual ao outro bot)
        if is_lootlabs(url):
            return await interaction.followup.send(
                f"https://baconbypass.xyz/lootlabs?url={url}"
            )

        start = time.perf_counter()
        try:
            async with aiohttp.ClientSession() as session:
                async with session.post(
                    f"{API}/bypass",
                    json={"url": url, "apikey": KEY},
                    timeout=aiohttp.ClientTimeout(total=60),
                ) as resp:
                    data = await resp.json(content_type=None)
                elapsed = time.perf_counter() - start
                remaining = await get_remaining(session)
        except Exception as e:
            print(e)
            return await interaction.followup.send(
                embed=error_embed("Error", "Could not reach the API. Try again later.")
            )

        if data.get("status") == "success":
            result = str(data.get("result"))
            embed = discord.Embed(title="Bypass Successful", color=GREEN)
            embed.add_field(name="Result", value=f"```{result[:1000]}```", inline=False)

            footer = f"Requested by {interaction.user.name} • Processed in {elapsed:.1f}s"
            if remaining is not None:
                footer += f" • {remaining} bypasses remaining today"
            embed.set_footer(text=footer, icon_url=interaction.user.display_avatar.url)

            return await interaction.followup.send(embed=embed, view=CopyView(result))

        msg = str(data.get("message", "Unknown error."))
        low = msg.lower()
        if "expired" in low:
            return await interaction.followup.send(
                embed=error_embed("Key Expired", "The API key has expired.")
            )
        if "invalid key" in low:
            return await interaction.followup.send(
                embed=error_embed("Invalid Key", "The API key is invalid.")
            )
        if "url" in low:
            return await interaction.followup.send(
                embed=error_embed("Invalid URL", "Please provide a valid URL.")
            )
        await interaction.followup.send(embed=error_embed("Error", msg[:1000]))


    @app_commands.command(name="supported", description="Lista os sites suportados")
    async def supported(self, interaction: discord.Interaction):
        await interaction.response.defer(ephemeral=True)
        try:
            async with aiohttp.ClientSession() as session:
                async with session.get(f"{API}/supported") as resp:
                    data = await resp.json(content_type=None)
        except Exception:
            return await interaction.followup.send(
                embed=error_embed("Error", "Could not reach the API. Try again later.")
            )

        sites = data.get("supported", [])
        text = "\n".join(f"• {s}" for s in sites)
        embed = discord.Embed(
            title=f"Supported sites ({data.get('total', len(sites))})",
            description=text[:4000] or "No data.",
            color=GREEN,
        )
        await interaction.followup.send(embed=embed)


async def setup(bot: commands.Bot):
    await bot.add_cog(Bypass(bot))


# --- Bot completo (rode este arquivo direto) ---
if __name__ == "__main__":
    import asyncio

    intents = discord.Intents.default()
    bot = commands.Bot(command_prefix="!", intents=intents)

    @bot.tree.error
    async def on_app_error(interaction: discord.Interaction, error):
        print("Erro no comando:", error)
        try:
            embed = error_embed("Error", "Something went wrong. Try again.")
            if interaction.response.is_done():
                await interaction.followup.send(embed=embed)
            else:
                await interaction.response.send_message(embed=embed)
        except Exception:
            pass

    @bot.event
    async def on_ready():
        await bot.tree.sync()
        print(f"Logado como {bot.user}")

    async def main():
        async with bot:
            await bot.add_cog(Bypass(bot))
            await bot.start(DISCORD_TOKEN)

    asyncio.run(main())
    
