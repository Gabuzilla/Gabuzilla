require("dotenv").config();
const axios = require("axios");
const fs = require("fs");

const REGION = "us";
const REALM_SLUG = "azralon";
const CHAR_NAME = "calltrucko";

async function getBlizzardToken() {
  const clientId = process.env.BLIZZARD_CLIENT_ID;
  const clientSecret = process.env.BLIZZARD_CLIENT_SECRET;
  const url = `https://${REGION}.battle.net/oauth/token`;
  try {
    const response = await axios.post(url, "grant_type=client_credentials", {
      auth: { username: clientId, password: clientSecret },
    });
    return response.data.access_token;
  } catch (error) {
    console.error(
      "Erro ao obter token da Blizzard:",
      error.response ? error.response.data : error.message
    );
    return null;
  }
}

async function getCharacterData(token) {
  const namespace = `profile-${REGION}`;
  const characterUrl = `https://${REGION}.api.blizzard.com/profile/wow/character/${REALM_SLUG}/${CHAR_NAME}?namespace=${namespace}&locale=pt_BR`;
  const mediaUrl = `https://${REGION}.api.blizzard.com/profile/wow/character/${REALM_SLUG}/${CHAR_NAME}/character-media?namespace=${namespace}&locale=pt_BR`;

  try {
    const [characterResponse, mediaResponse] = await Promise.all([
      axios.get(characterUrl, {
        headers: { Authorization: `Bearer ${token}` },
      }),
      axios.get(mediaUrl, { headers: { Authorization: `Bearer ${token}` } }),
    ]);

    const charData = characterResponse.data;
    const mediaData = mediaResponse.data;

    const mainImage = mediaData.assets.find(
      (asset) => asset.key === "main-raw"
    );

    return {
      level: charData.level,
      ilvl: charData.equipped_item_level,
      guild: charData.guild ? charData.guild.name : "Sem Guilda",
      spec: charData.active_spec.name,
      charClass: charData.character_class.name,
      faction: charData.faction.name,
      achievements: charData.achievement_points,
      imageUrl: mainImage ? mainImage.value : null,
    };
  } catch (error) {
    console.error(
      "error:",
      error.response ? error.response.data : error.message
    );
    return null;
  }
}

function updateReadme(characterData) {
  const readmePath = "./README.md";
  let readmeContent;
  try {
    readmeContent = fs.readFileSync(readmePath, "utf8");
  } catch (error) {
    console.error("Error reading README.md:", error);
    return;
  }

  const characterMarkdown = `
## 🎮 currently playing wow
  <div align="center">
  <img src="${characterData.imageUrl}" alt="${CHAR_NAME}" width="650px" />
  <table >
    <tr>
      <td><strong>Level:</strong> ${characterData.level}</td>
      <td><strong>Item Level:</strong> ${characterData.ilvl}</td>
      <td><strong>Class:</strong> ${characterData.charClass} (${characterData.spec})</td>
    </tr>
  </table>
</div>
`;

  const updatedReadme = readmeContent.replace(
    /(<!-- WOW-STATUS-START -->)[\s\S]*(<!-- WOW-STATUS-END -->)/,
    `$1\n${characterMarkdown}\n$2`
  );

  try {
    fs.writeFileSync(readmePath, updatedReadme, "utf8");
    console.log("README updated successfully!");
  } catch (error) {
    console.error("Error saving README.md:", error);
  }
}

async function main() {
  const token = await getBlizzardToken();
  if (token) {
    const characterData = await getCharacterData(token);
    if (characterData) {
      updateReadme(characterData);
    }
  }
}

main();
