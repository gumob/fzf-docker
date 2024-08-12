### Execute docker command
function fzf-docker() {
  # $1 (Optionally) - if present (value doesn't matter), this test checks if any containers exist. Else, if active (running) containers exis
  function __docker_pre_test() {
    if [[ -z "$1" ]] && [[ $(docker ps --format '{{.Names}}') ]]; then
      return 0
    fi

    if [[ ! -z "$1" ]] && [[ $(docker ps -a --format '{{.Names}}') ]]; then
      return 0
    fi

    echo "No containers found"
    return 1
  }

  function __docker_images_pre_test() {
    if [[ $(docker images -qa) ]]; then
      return 0
    fi
    echo "No images found"
    return 1
  }

  #1 (Optional) path to `docker-compose.yml` file
  function __docker_compose_pre_test() {
    if [ -z "$1" ]; then
      if [[ $(docker-compose config --services) ]]; then
        return 0
      fi
      echo "No docker-compose.yml found (or it contains errors). You can pass as the first argument a path to the service declaration file."
      return 1
    fi

    if [[ $(docker-compose --file $1 config --services) ]]; then
      return 0
    fi
    echo "Invalid service declaration file $1."
    return 1
  }

  # $1: time interval - e.g.: `1m` for 1 minute
  # $2: names of container(s) to display logs from
  function __docker_logs() (
    local since=""
    if [ ! -z "$1" ]; then
      since="--since $1 "
    fi

    local count=$(wc -l <<<$2)
    if [[ -z "$2" ]]; then
      return 1
    fi
    if [[ "$count" -eq "1" ]]; then
      eval "docker logs -f $since$2"
      return 0
    fi

    local resetColor="\x1b[39m\x1b[49m"
    #list of 48 distinct colors
    local allColors="\x1b[90m\n\x1b[92m\n\x1b[93m\n\x1b[94m\n\x1b[95m\n\x1b[96m\n\x1b[97m\n\x1b[30m\n\x1b[31m\n\x1b[32m\n\x1b[33m\n\x1b[34m\n\x1b[35m\n\x1b[36m\n\x1b[40m\x1b[90m\n\x1b[40m\x1b[91m\n\x1b[40m\x1b[92m\n\x1b[40m\x1b[94m\n\x1b[40m\x1b[95m\n\x1b[40m\x1b[96m\n\x1b[40m\x1b[97m\n\x1b[41m\x1b[90m\n\x1b[41m\x1b[93m\n\x1b[41m\x1b[95m\n\x1b[41m\x1b[97m\n\x1b[42m\x1b[90m\n\x1b[42m\x1b[93m\n\x1b[42m\x1b[97m\n\x1b[43m\x1b[90m\n\x1b[43m\x1b[93m\n\x1b[43m\x1b[97m\n\x1b[44m\x1b[91m\n\x1b[44m\x1b[92m\n\x1b[44m\x1b[93m\n\x1b[44m\x1b[95m\n\x1b[44m\x1b[97m\n\x1b[45m\x1b[93m\n\x1b[45m\x1b[97m\n\x1b[46m\x1b[90m\n\x1b[46m\x1b[91m\n\x1b[46m\x1b[92m\n\x1b[46m\x1b[93m\n\x1b[46m\x1b[96m\n\x1b[46m\x1b[97m\n\x1b[47m\x1b[90m\n\x1b[47m\x1b[95m\n\x1b[47m\x1b[96m\n"
    #list of `$count` number of distinct colors
    local colors=$(echo -e "$allColors" | shuf -n $count)

    local allPids=()
    local writeToTmpFilePids=()
    local tmpFile="/tmp/fzf-docker-logs-$(date +'%s')"

    function _exit {
      for pid in "${allPids[@]}"; do
        # ignore if process is not alive anymore
        kill -9 $pid >/dev/null 2>/dev/null
      done

      test -e $tmpFile && rm -f $tmpFile
    }
    trap _exit INT TERM SIGTERM

    while read -r name; do
      # last color from list
      local color=$(echo -e "$colors" | tail -n 1)
      # update list - remove last color from list
      colors=$(echo -e "$colors" | head -n -1)

      # in bash, to get the pid for `docker logs` (so we can kill it in _exit), use `command1 > >(command2)` instead of `command1 | command2` - see https://stackoverflow.com/a/8048493/2732818
      # sed -u needed as explained in https://superuser.com/a/792051
      eval "docker logs --timestamps -f $since\"$name\" 2>&1 > >(sed -u -e \"s/^/${color}[${name}]${resetColor} /\" >> $tmpFile) &"
      local pid=($!)

      allPids+=($pid)
      writeToTmpFilePids+=($pid)
    done <<<"$2" # bash executes while loops in pipe in subshell, meaning pids will not be available outside of loop when using `echo -e "$2" | while...`

    #wait for all historc logs being written into $tmpFile
    sleep 2

    local removeTimestamp='sed -r -u "s/((\x1b\[[0-9]{2}m){0,2}\[.*\]\x1b\[39m\x1b\[49m )[^ ]+ /\1/"'

    #sort historic logs
    local numOfLines=$(wc -l <$tmpFile)
    eval "head -n $numOfLines $tmpFile | sort --stable --key=2 | $removeTimestamp"

    #show new logs
    local numOfLines=$((numOfLines + 1))
    #2>/dev/null because "tail: /tmp/fzf-docker-logs: file truncated" is outputed every time $tmpFile is emptied
    eval "tail -f -n +$numOfLines $tmpFile > >($removeTimestamp) 2>/dev/null &"
    allPids+=($!)

    #we don't really need to keep the logs on the hdd in $tmpFile so every minute empty it. But keep one line, so the "tail -f" can keep track
    eval "while true; do tail -n 1 $tmpFile > $tmpFile; sleep 10s; done &"
    allPids+=($!)

    #wait for all docker containers have stoped, i.e. no more logs can be generated
    for pid in "${writeToTmpFilePids[@]}"; do
      wait $pid
    done

    #this also kills the `tail -f $tmpFile` process
    _exit
  )

  function __docker_compose_fileref_generator() {
    if [ ! -z "$1" ]; then
      echo "--file $1"
    fi
  }

  #1 (Optional) path to `docker-compose.yml` file
  #2 A regexp that needs to be present in the docker-compose.yml file for this command to return it
  function __docker_compose_parse_services_config() {
    local fileref=$(__docker_compose_fileref_generator $1)

    # `docker-compose config` normalizes the indentation and format, no matter what the actual file is...
    local config=$(eval "docker-compose $fileref config")

    local inServicesBlock='false'
    local service=''

    echo -e "$config" |
      while IFS= read -r line; do
        if [[ "$line" == "services:" ]]; then
          inServicesBlock='true'
        elif [[ "$line" =~ ^[^:" "]+:.*$ ]]; then
          inServicesBlock='false'
        fi

        if [[ "$inServicesBlock" == "true" ]]; then
          if [[ "$line" =~ ^" "{2}[^:" "]+:$ ]]; then
            local service=$(echo -e "$line" | grep -o '[^: ]\+')
          fi

          if [[ "$line" =~ $2 ]] && [[ ! -z "service" ]]; then
            echo "$service"
            service=''
          fi
        fi
      done
  }

  # docker restart
  function docker-restart() {
    __docker_pre_test
    if [ $? -eq 0 ]; then
      local containers=$(docker ps --format '{{.Names}}' | fzf --prompt="docker restart > " -m)

      echo -e "$containers" |
        while read -r name; do
          echo "Restarting $name..."
          docker restart $name
        done

      __docker_logs "1m" "$containers"
    fi
  }

  # docker logs
  function docker-logs() {
    __docker_pre_test "all"
    if [ $? -eq 0 ]; then
      local containers=$(docker ps -a --format '{{.Names}}' | fzf --prompt="docker logs > " -m)
      __docker_logs "$1" "$containers"
    fi
  }

  # docker logs all
  function docker-logs-all-containers() {
    __docker_pre_test "all"
    if [ $? -eq 0 ]; then
      local containers=$(docker ps -a --format '{{.Names}}')
      __docker_logs "$1" "$containers"
    fi
  }

  # docker exec
  function docker-exec() {
    __docker_pre_test
    if [ $? -eq 0 ]; then
      local name=$(docker ps --format '{{.Names}}' | fzf --prompt="docker exec > ")

      if [ ! -z "$name" ]; then
        local command="$1"

        if [ -z "$command" ] && [ -f "$HOME/.fzfdocker-exec" ]; then
          command=$($HOME/.fzfdocker-exec "$name")
        fi

        if [ -z "$command" ]; then
          local imageName=$(docker inspect --format '{{.Config.Image}}' $name | sed -e 's/:.*$//g') #without version
          case "$imageName" in
          "mysql" | "bitnami/mysql" | "mysql/mysql-server" | "percona" | centos/mysql*)
            command='mysql -uroot -p$MYSQL_ROOT_PASSWORD'
            ;;

          "mongo" | "circleci/mongo")
            command='if [ -z "$MONGO_INITDB_ROOT_USERNAME" ]; then mongo; else mongo -u "$MONGO_INITDB_ROOT_USERNAME" -p "$MONGO_INITDB_ROOT_PASSWORD"; fi'
            ;;
          "bitnami/mongodb")
            command='if [ ! -z "$MONGODB_ROOT_PASSWORD" ]; then mongo -u root -p "$MONGODB_ROOT_PASSWORD"; elif [ ! -z $MONGODB_USERNAME ]; then mongo -u "$MONGODB_USERNAME" -p "$MONGODB_PASSWORD" "$MONGODB_DATABASE"; else mongo; fi'
            ;;
          centos/mongodb*)
            command='mongo -u "admin" -p "$MONGODB_ADMIN_PASSWORD" --authenticationDatabase admin'
            ;;

          "redis" | "circleci/redis" | "bitnami/redis" | centos/redis*)
            command='echo -n "Enter DB Number to connect to (^[1-9][0-9]?$): " && read dbNum && redis-cli -n $dbNum'
            if [[ "$imageName" == "bitnami/redis" ]] || [[ "$imageName" == "centos/redis"* ]]; then
              command="if [ -z \$REDIS_PASSWORD ]; then $command; else $command -a \"\$REDIS_PASSWORD\"; fi"
            fi
            ;;

          *)
            command='command $(command -v zsh || command -v bash || command -v ash || command -v sh)'
            ;;
          esac

          command="sh -c '$command'"
        fi

        eval "docker exec -it $name $command"
      fi
    fi
  }

  # docker remove
  function docker-remove-container() {
    __docker_pre_test "all" &&
      docker ps -aq --format "{{.Names}}" |
      fzf -m |
        while read -r name; do
          docker rm -f $name
        done
  }

  # docker remove all
  function docker-remove-all-container() {
    __docker_pre_test "all" &&
      docker rm $(docker ps -aq) -f
  }

  # docker stop
  function docker-stop() {
    __docker_pre_test &&
      docker ps --format '{{.Names}}' |
      fzf -m |
        while read -r name; do
          docker update --restart=no $name
          docker stop $name
        done
  }

  # docker stop all
  function docker-stop-all-running-container() {
    __docker_pre_test
    if [ $? -eq 0 ]; then
      docker update --restart=no $(docker ps -q)
      docker stop $(docker ps -q)
    fi
  }

  # docker stop
  function docker-stop-and-remove-container() {
    __docker_pre_test &&
      docker ps --format '{{.Names}}' |
      fzf -m |
        while read -r name; do
          docker update --restart=no $name
          docker stop $name
          docker rm -f $name
        done
  }

  # docker stop all
  function docker-stop-and-remove-all-containers() {
    docker-stop-all-running-container

    docker-remove-all-container
  }

  # docker kill
  function docker-kill() {
    __docker_pre_test &&
      docker ps --format '{{.Names}}' | fzf --prompt="docker kill > " -m --print0 |
      fzf -m |
        while read -r name; do
          docker update --restart=no $name
          docker kill $name
        done
  }

  # docker kill
  function docker-kill-all-containers() {
    __docker_pre_test
    if [ $? -eq 0 ]; then
      docker update --restart=no $(docker ps -q)
      docker kill $(docker ps -q)
    fi
  }

  # docker kill
  function docker-kill-and-remove-container() {
    __docker_pre_test &&
      docker ps --format '{{.Names}}' |
      fzf -m |
        while read -r name; do
          docker update --restart=no $name
          docker kill $name
          docker rm -f $name
        done
  }

  # docker kill
  function docker-kill-and-remove-all-containers() {
    docker-kill-all-containers

    docker-remove-all-container
  }

  # docker remove image
  function docker-remove-image() {
    __docker_images_pre_test
    if [ "$?" -eq "0" ]; then
      echo "Loading images. Depending on the number of images you have, this may take a while..."

      #column1: comma seperated list of ids to be removed if this option is selected
      #column2: text to be displayed to the user.
      #seperator: tab ("\t")
      local images=''

      # create image references for non-dangling images
      for id in $( #remove duplicate ids. necessary because each image tag has an entry, and images may have multiple tags
        docker images --format "{{.ID}}" --filter "dangling=false" | sort | uniq
      ); do
        local references=$(docker inspect "$id" --format "{{range .RepoTags}}{{.}}\n{{end}}{{range .RepoDigests}}{{.}}\n{{end}}")

        local list=$(echo "$references" | sed 's/\\n$//' | sed -r 's/\\n/\n/g') #remove trailing new line && replace \n with actual new line
        local listLength=$(echo "$list" | wc -l)
        local head=$(echo "$list" | head -n 1)
        local tail=$(echo "$list" | tail -n 1)

        local result="$id\t"
        for ref in $(echo -e "$list"); do
          if [[ "$head" == "$ref" ]]; then
            result+="$ref"
            if [[ "$listLength" == "2" ]]; then
              result+=" (alias: "
            elif [[ "$listLength" != "1" ]]; then
              result+=" (aliases: "
            fi
          elif [[ "$tail" == "$ref" ]]; then
            result+="$ref)"
          else
            result+="$ref, "
          fi
        done

        if [[ "$images" == "" ]]; then
          images="$result"
        else
          images+="\n$result"
        fi
      done

      # create image references for dangling images
      local ids=$(docker images --format "{{.ID}}" --filter "dangling=true")
      local numberOfIds=$(echo "$ids" | wc -l)
      if [[ "$numberOfIds" != "0" ]]; then
        #add option to remove all dangling images
        local allDanglingImages=$(echo -e "$ids" | sed ':a; N; s/\n/,/; ta') # join all ids into CSV list
        allDanglingImages+="\tAll dangling images (total of $numberOfIds)"
        if [[ "$images" == "" ]]; then
          images="$allDanglingImages"
        else
          images+="\n$allDanglingImages"
        fi

        #add options, group by label
        local idCreatedLabel=''
        for id in $(echo "$ids" | head -n 20); do
          local created=$(docker inspect "$id" --format "{{.Created}}")
          local labels=$(docker inspect "$id" --format '{{ range $key, $value := .Config.Labels }}{{ $key }}:{{$value}}\n{{end}}')

          while read -r label; do #also entered for no label
            local reference=$(echo -e "$id\t$created\t$label")
            if [[ "$idCreatedLabel" == "" ]]; then
              idCreatedLabel="$reference"
            else
              idCreatedLabel+="\n$reference"
            fi
          done <<<"$(echo "$labels" | sed 's/\\n$//' | sed -r 's/\\n/\n/g')" #remove trailing new line && replace \n with actual new line
        done

        #$1=id
        #$2=created date
        #$3=label
        #a = all ids that belong to label
        #b = max created date
        #c = max created id
        # shellcheck disable=SC2016
        local awk='
        {
          a[$3] = a[$3] ? a[$3]","$1 : $1; #create a CSV list of ids (grouped by label)

          if( !b[$3] || b[$3] < $2 ) { #if the current row created date is bigger than all dates in previous rows
            b[$3] = $2; #set date
            c[$3] = $1; #set id
          }
        } END {
          for ( i in a ) {
            print (i ? i : "none") #replace no label with "none" here, as otherwise the splitting of columns didnt work below
            "\t"
            a[i]
            "\t"
            c[i];
          }
        }'
        while read -r labelIdsLatestId; do
          local label="$(echo -e "$labelIdsLatestId" | cut -f 1)"
          local allIds="$(echo -e "$labelIdsLatestId" | cut -f 2)"
          local numberOfIds="$(echo "$allIds" | awk -F ',' '{print NF}')"
          local latestBuildId="$(echo -e "$labelIdsLatestId" | cut -f 3)"

          local labelReference=""
          if [[ "$label" == "none" ]]; then #see awk. since labels always contain key and value (seperated by a colon), this is safe
            labelReference='that have no label'
          else
            labelReference="that have the label $label"
          fi

          local imagesPluralOrSingular="image"
          if [[ "$numberOfIds" != "1" ]]; then
            imagesPluralOrSingular+="s"
          fi

          images+="\n$allIds\tAll dangling $imagesPluralOrSingular (total of $numberOfIds) $labelReference"

          if [[ "$numberOfIds" != "1" ]]; then
            local idsWithoutLatest="$(echo "$allIds" | sed -r "s/(^$latestBuildId,|,$latestBuildId)//")"

            imagesPluralOrSingular="image"
            if [[ "$numberOfIds" != "2" ]]; then #2 because the number of ids still includes the id to keep (newest created date)
              imagesPluralOrSingular+="s"
            fi

            images+="\n$idsWithoutLatest\tAll EXCEPT THE NEWEST dangling $imagesPluralOrSingular (total of $((numberOfIds - 1))) $labelReference"
          fi
        done <<<"$(echo -e "$idCreatedLabel" | awk -F "\t" "$awk")"
      fi

      #column1 contains ids. multiple ids per line may be comma seperated
      #because there may be an overlap in dangling image ids, remove duplicates.
      echo -e "$images" |
        fzf -m -d "\t" --with-nth=2 |
        cut -f 1 |
        sed 's/,/\n/g' |
        sort |
        uniq |
        while read -r idToRemove; do
          docker rmi "$idToRemove"
        done
    fi
  }

  # docker remove all images
  function docker-remove-all-images() {
    __docker_images_pre_test &&
      docker rmi $(docker images -qa) -f
  }

  # docker clean
  function docker-stop-and-remove-all-containers_docker-remove-all-images() {
    docker-stop-and-remove-all-containers
    docker-remove-all-images
  }

  # docker compose up
  function docker-compose-up() {
    local fileref=$(__docker_compose_fileref_generator $1)

    __docker_compose_pre_test $1 &&
      eval "docker-compose $fileref config --services" |
      fzf -m |
        while read -r service; do
          eval "docker-compose $fileref up -d $service"
        done
  }

  # docker compose up all
  function docker-compose-up-all-services() {
    local fileref=$(__docker_compose_fileref_generator $1)

    __docker_compose_pre_test $1 &&
      eval "docker-compose $fileref up -d"
  }

  # docker compose build
  function docker-compose-build() {
    local fileref=$(__docker_compose_fileref_generator $1)

    __docker_compose_pre_test $1 &&
      __docker_compose_parse_services_config "$1" '^    build:.*$' |
      fzf -m |
        while read -r service; do
          eval "docker-compose $fileref build --force-rm --no-cache --pull $service"
        done
  }

  # docker compose build all
  function docker-compose-build-all() {
    local fileref=$(__docker_compose_fileref_generator $1)

    __docker_compose_pre_test $1 &&
      eval "docker-compose $fileref build --force-rm --no-cache --pull --parallel"
  }

  # docker compose pull
  function docker-compose-pull() {
    local fileref=$(__docker_compose_fileref_generator $1)

    __docker_compose_pre_test $1 &&
      __docker_compose_parse_services_config "$1" '^    image:.*$' |
      fzf -m |
        while read -r service; do
          eval "docker-compose $fileref pull --ignore-pull-failures $service"
        done
  }

  # docker compose pull all
  function docker-compose-pull-all-services() {
    local fileref=$(__docker_compose_fileref_generator $1)

    __docker_compose_pre_test $1 &&
      eval "docker-compose $fileref pull --ignore-pull-failures"
  }

  # docker compose update image
  function docker-compose-update-image() {
    local fileref=$(__docker_compose_fileref_generator $1)

    __docker_compose_pre_test $1
    if [ "$?" -eq "0" ]; then
      local buildServices=$(__docker_compose_parse_services_config "$1" '^    build:.*$')
      local pullServices=$(__docker_compose_parse_services_config "$1" '^    image:.*$')

      eval "docker-compose $fileref config --services" |
        fzf -m |
        while read -r service; do
          echo -e "$buildServices" |
            while read -r buildService; do
              if [[ "$buildService" == "$service" ]]; then
                eval "docker-compose $fileref build --force-rm --no-cache --pull $service"
                continue 2
              fi
            done

          echo -e "$pullServices" |
            while read -r pullService; do
              if [[ "$pullService" == "$service" ]]; then
                eval "docker-compose $fileref pull --ignore-pull-failures $service"
                continue 2
              fi
            done
        done
    fi
  }

  # docker compose update image all services
  function docker-compose-build-all_docker-compose-pull-all-services() {
    docker-compose-build-all "$1"

    docker-compose-pull-all-services "$1"
  }

  function fzf-docker-debug-info() {
    location="${BASH_SOURCE[0]}"
    if [ -z "$location" ]; then #fzf
      location=$(type -a $0 | sed "s/$0 is a shell function from //g")
    fi
    gitRepo="$(dirname $location)/.git"
    commitHash=$(eval "git --git-dir $gitRepo rev-parse HEAD")
    latestTag=$(eval "git --git-dir $gitRepo describe --tags")
    fzfDockerVersion="$latestTag ($commitHash)"

    shellEnvironment=$(ps p "$$" o cmd=)

    shell=$(echo $shellEnvironment | grep -oE "^\S+")
    shellVersion=$(eval "$shell --version")
    fzfVersion=$(fzf --version)

    dockerVersion=$(docker --version)
    dockerComposeVersion=$(docker-compose --version)

    dockerExecCustomDefaults=$(test -f "$HOME/.fzfdocker-exec" && cat "$HOME/.fzfdocker-exec" || echo "N.A.")

    for v in "fzfDockerVersion" "shellEnvironment" "shellVersion" "fzfVersion" "dockerVersion" "dockerComposeVersion" "dockerExecCustomDefaults"; do
      echo -e "${v}:"
      v=$(eval "echo -e \"\$$v\"" | sed 's/^/\t/g')
      echo -e "$v"
    done
  }


  # Define the command list with groups and commands
  local option_list=(
    "docker restart && open logs (in follow mode)"
    "docker logs (in follow mode)"
    "docker logs (in follow mode) all containers"
    "docker exec in interactive mode"
    "docker remove container (with force)"
    "docker remove all containers (with force)"
    "docker stop"
    "docker stop all running containers"
    "docker stop and remove container"
    "docker stop and remove all containers"
    "docker kill"
    "docker kill all containers"
    "docker kill and remove container"
    "docker kill and remove all containers"
    "docker remove image (with force). This includes options to remove dangling images."
    "docker remove all images (with force). This includes dangling images."
    "docker stop and remove all containers && docker remove all images"
    "docker-compose up (in detached mode)"
    "docker-compose up all services (in detached mode)"
    "docker-compose build (with --no-cache and --pull)"
    "docker-compose build (with --no-cache and --pull) all"
    "docker-compose pull"
    "docker-compose pull all services"
    "docker-compose update image (rebuild or pull)"
    "docker-compose build (with --no-cache and --pull) all && docker-compose pull all services"
  )

  # Define the corresponding BUFFER values
  local command_list=(
    "docker-restart" "docker-logs" "docker-logs-all-containers" "docker-exec"
    "docker-remove-container" "docker-remove-all-container" "docker-stop"
    "docker-stop-all-running-container" "docker-stop-and-remove-container"
    "docker-stop-and-remove-all-containers" "docker-kill" "docker-kill-all-containers"
    "docker-kill-and-remove-container" "docker-kill-and-remove-all-containers" "docker-remove-image"
    "docker-remove-all-images" "docker-stop-and-remove-all-containers_docker-remove-all-images"
    "docker-compose-up" "docker-compose-up-all-services" "docker-compose-build" "docker-compose-build-all"
    "docker-compose-pull" "docker-compose-pull-all-services" "docker-compose-update-image"
    "docker-compose-build-all_docker-compose-pull-all-services"
  )

  # Use fzf to select a command
  selected_option=$(printf "%s\n" "${option_list[@]}" | fzf --prompt="docker > ")
  # Extract the command name
  if [ -n "$selected_option" ]; then
    index=$(printf "%s\n" "${option_list[@]}" | grep -nFx "$selected_option" | cut -d: -f1)
    if [ -n "$index" ]; then
      if [[ "$SHELL" == *"/bin/zsh" ]]; then
        BUFFER="${command_list[$index]}"
      elif [[ "$SHELL" == *"/bin/bash" ]]; then
        BUFFER="${command_list[$index-1]}"
      else
        echo "Error: Unsupported shell. Please use bash or zsh to use fzf-docker."
        return 1
      fi
      zle accept-line
    fi
  fi
}


zle -N fzf-docker
if [[ "$SHELL" == *"/bin/zsh" ]]; then
  bindkey "${FZF_DOCKER_KEY_BINDING}" fzf-docker
elif [[ "$SHELL" == *"/bin/bash" ]]; then
  bind -x "'${FZF_DOCKER_KEY_BINDING}': fzf_docker"
fi

